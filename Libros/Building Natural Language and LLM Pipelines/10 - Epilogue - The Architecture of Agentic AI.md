---
title: 10 - Epilogue - The Architecture of Agentic AI
libro: Building Natural Language and LLM Pipelines
autor: deepset / Packt
capitulo: 10
created: 2026-06-12
tags:
  - libros/building-natural-language-and-llm-pipelines
  - type/case-study
  - status/permanent
aliases:
  - Epilogue - The Architecture of Agentic AI
  - Cap 10 - Epilogue
updated: 2026-06-12
---

# 10 - Epilogue - The Architecture of Agentic AI

> [!info] Capítulo 10 (Epílogo) · *Building Natural Language and LLM Pipelines* — Packt (ISBN 9781835467992)
> La línea de llegada del libro: formalizar la tesis **tool-vs-orchestration** en un blueprint definitivo y demostrar empíricamente —vía tres arquitecturas del **Yelp Navigator** (V1/V2/V3)— que *la fiabilidad NO es propiedad del modelo sino de la arquitectura que lo rodea*. Desacoplar el *doing* ([[Haystack 2.0]]/[[Hayhooks]], tool layer determinista) del *thinking* ([[LangGraph]], orchestration layer stateful), envolver modelos open-weight en guardrails/retries/persistencia, y tratar al LLM como un componente no fiable dentro de un sistema fiable = aplicar **[[Site Reliability Engineering (SRE)]] a la IA**. El resultado son **[[Sovereign agent|sovereign agents]]** que corren localmente y degradan gracefully en vez de alucinar. Navegá: [[_Building Natural Language and LLM Pipelines|Building NL & LLM Pipelines]] · anterior [[09 - Future Trends and Beyond]].

## Resumen

El Epílogo cierra conceptualmente toda la obra: lo que en el [[01 - Introduction to Natural Language Processing Pipelines|Cap 1]] se planteó como la **agentic reliability crisis** se resuelve aquí con un blueprint demostrable. La tesis central es que hemos llegado a un **agentic inflection point**: el gap entre lo que un modelo *puede hacer* (capability) y lo que *podemos controlar* (governance). En la era generativa 2023-2025 el foco fue la capability —un ciclo de escalada entre closed y open-weight models— pero al pasar de pilots aislados a deployments enterprise, la naturaleza probabilística de los LLMs choca con los requisitos deterministas del negocio. La resolución NO es prompt engineering, sino la disciplina formal de **[[Context Engineering|context engineering]]** (write/select/compress/isolate) más arquitecturas graph-based que imponen integridad estructural alrededor del reasoning.

El núcleo del capítulo es la **tesis tool-versus-orchestration**: desacoplar el *doing* del *thinking* en dos capas con garantías de fiabilidad y failure modes distintos. La **tool layer** son pipelines [[Haystack 2.0|Haystack]] robustos desplegados como microservicios vía [[Hayhooks]] —directed graphs deterministas que el agente trata como **deterministic black boxes** vía `HTTP POST`—; la **orchestration layer** es una state machine [[LangGraph]] que gestiona reasoning fluido y stateful. Sobre cinco building blocks (State, Prompts, Tools, Nodes, Graph) el libro construye y compara tres versiones del Yelp Navigator: **V1** (sequential chain, el *shallow agent* construido sobre el happy path), **V2** (router/[[Supervisor-worker pattern|supervisor]] con clarification step y structured state), y **V3** (resilient supervisor con guardrails node determinista, `RetryPolicy`, [[Circuit breaker|circuit breaker]] y persistencia/memory).

La demostración es doble. El **argumento cuantitativo** (Tabla 10.1) muestra que V2 reduce tokens hasta un **74%** vs V1 al no re-leer el JSON entero en cada paso del supervisor. El **case study de microservice failure** (Tablas 10.2-10.4) apaga intencionalmente los tools y testea tres modelos open-weight (**GPT-OSS 20B, [[DeepSeek-R1]], Qwen 3**): V1/V2 caen en *recursion traps* y *deep retries* donde el summary node **alucina** businesses inexistentes con phone numbers y websites, mientras V3 dispara un *safe exit* el **100% de las veces**. El insight: el único node que alucinó fue el constreñido por la naturaleza subjetiva del prompt; los constreñidos por código Python o pipelines Haystack no cedieron. La conclusión —reforzada por casos reales como el NYC legal brief (2023), Air Canada (2024) y el multi-agent loop de $47K (2025)— es que **orchestration outweighs intelligence**. Finalmente, las cuatro estrategias de context engineering se mapean a V3, y todo converge en el **sovereign stack**: ingestion local (Haystack) + inference local (Ollama/vLLM) + state local (LangGraph), económicamente viable por su marginal cost cercano a cero y obligatorio para data sensible. El cierre deja cuatro principios para el futuro: tool-vs-orchestration, data-centricity over prompting, the sovereign stack, y engineering integrity (tratar al LLM como un **Stochastic Processing Unit**).

## The agentic inflection point

El conflicto central del capítulo es el **gap entre lo que un modelo puede hacer y lo que podemos controlar**. En la era generativa 2023-2025 el foco fue la **capability**, en un ciclo de escalada entre closed models y open-weight: **GPT-4** y **Claude 3 Opus** empujaron el general reasoning, **Gemini 1.5** y **Gemini 3** la context retention masiva, y **Llama 3** y **[[DeepSeek-R1]]** trajeron inteligencia frontier a infraestructura local y soberana. Pero al pasar de pilots aislados a deployments enterprise (2025+), la preocupación deja de ser capability y pasa a ser **governance**.

> [!note] El **agentic inflection point** es el punto donde la naturaleza **probabilística** de los LLMs choca con los requisitos **deterministas** del negocio. Ese choque es la **agentic reliability crisis** planteada en el [[01 - Introduction to Natural Language Processing Pipelines|Cap 1]].

La resolución NO es prompt engineering, sino dos cosas combinadas: la disciplina formal de **[[Context Engineering|context engineering]]** y las **arquitecturas graph-based** que imponen integridad estructural alrededor del reasoning.

> [!note] **Context engineering** = la disciplina sistemática del diseño y gestión del **entorno de información** provisto al LLM en inference time. Está gobernada por cuatro estrategias: **write, select, compress, isolate** (introducidas en el [[02 - Diving Deep into Large Language Models|Cap 2]]). El **prompt engineering** se ocupa de *cómo pedirle* al modelo; el **context engineering** de *qué información ve* el modelo y cómo se estructura.

Las **hallucinations** son la barrera de fondo y cambian de naturaleza según el sistema:

- Cuando un agente es un chatbot **sin infraestructura**, las hallucinations son un **inconvenience**.
- Cuando el agente corre **APIs y tools** para tomar decisiones, las hallucinations se vuelven **liabilities**.

> [!warning] Las early architectures eran simples **tool-calling loops** —"shallow agents"— que fallan ante las realidades modernas. La idea clave: la reliability y la performance NO deben depender solo de los LLMs, sino **coexistir en un sistema cuidadosamente ingenierizado**.

La salida que propone el libro: envolver **reasoning models open-weight** (**GPT-OSS, [[DeepSeek-R1]], Qwen 3**) en una estructura de control resiliente (**[[LangGraph]]**) y groundearlos con pipelines deterministas (**[[Haystack 2.0]]**) → **[[Sovereign agent|sovereign agents]]**: sistemas que operan confiablemente en infra local, independientes de APIs centralizadas. El proyecto Yelp Navigator evoluciona V1→V3 (código en `epilogue/context-engineering`).

## The tool versus orchestration thesis

Es la tesis arquitectónica central del libro (planteada en el [[01 - Introduction to Natural Language Processing Pipelines|Cap 1]], refinada en el [[08 - Hands-On Projects|Cap 8]]): **desacoplar el *doing* (tool layer) del *thinking* (orchestration layer)**. En las etapas tempranas iban juntas —se le pedía al LLM razonar Y ejecutar la lógica de retrieval en el mismo ciclo de generación— lo que produce fragilidad:

> [!warning] Cuando el reasoning y el fetching viven en el mismo ciclo, **el reasoning engine se distrae con la mecánica del fetching, y el fetching se corrompe por la variabilidad del reasoning**. Las tres arquitecturas (V1/V2/V3) separan formalmente estas capas, tratándolas como capas distintas con distintas garantías de fiabilidad y failure modes.

### The tool layer: Haystack as the engine of reliability

Los tools NO son funciones Python simples ni wrappers de API crudos: son **pipelines robustos con [[Haystack 2.0]] desplegados como microservicios vía [[Hayhooks]]**. La diferencia es estructural: un tool estándar llama `requests.get()` a data cruda; un pipeline Haystack es un **directed graph (DG)** que encapsula toda la complejidad de retrieval, preprocessing, embedding y reranking.

> [!note] Un pipeline Haystack impone **procesamiento determinista**: o tiene éxito o falla, pero NO alucina una estrategia de retrieval. El agente NO ejecuta el código del RAG pipeline directamente —hace `HTTP POST` requests a un endpoint local.

El código del tool (un único `tools.py` compartido entre las 3 arquitecturas):

```python
@tool
def search_businesses(query: str) -> Dict[str, Any]:
"""Search for businesses using natural language query."""
    try:
        response = requests.post(f"{BASE_URL}/business_search/run",
            json={"query": query}, timeout=30)
```

La **isolation** tiene implicaciones de estabilidad concretas:

- El RAG pipeline **escala independientemente** del agente.
- Se **actualiza sin redeployar** el agente.
- Mantiene sus **dependencias pesadas** (PyTorch, vector DB drivers) separadas del agent runtime.

> [!tip] El agente trata el RAG como una **deterministic black box**: solo necesita su signature y saber *cuándo* usarlo. Los microservicios Haystack manejan **data processing rígido y high-throughput**; el orquestador LangGraph maneja **reasoning fluido y stateful**.

### The orchestration layer: LangGraph as the cognitive architecture

Si Haystack es el **cuerpo** del agente (los músculos y órganos sensoriales que interactúan con el mundo), [[LangGraph]] es el **cerebro**. Los pipelines estándar luchan con **cyclical logic (loops)** y **state persistence**, ambos prerrequisitos del advanced reasoning.

> [!note] **[[LangGraph]] 1.0** introduce una **state machine architecture**: los **nodes** (agents/functions) modifican un **shared persistent state**, y los **edges** (transitions) se determinan dinámicamente.

Los patrones de orquestación por versión:

- **V1** = **sequential**: los agentes ejecutan y eligen el next node; un final node aprueba el summary.
- **V2 y V3** = separan **ejecución de decisión** vía el **[[Supervisor-worker pattern|supervisor-worker pattern]]**: un **supervisor node** central NO hace el trabajo —es un **meta-cognitive controller** que evalúa el state, crea un plan y delega a worker nodes especializados (los Hayhooks tools).

> [!warning] V2 y V3 comparten state y graph structure similares, pero la diferencia decisiva es el supervisor: **V2 = supervisor "naive"** (sin guardrails, fail-safes ni persistencia) y **V3 = supervisor "resilient"** (con esas tres features).

## Tracing the evolution of architecture

El Yelp Navigator es un **microcosmos de la maduración del campo** entre 2023 y 2025: de scripts naive a software engineering robusto. Todo sistema ágentico se construye con **cinco building blocks**:

| Building block | Qué es |
|---|---|
| **State** | La data structure compartida = **memoria del sistema**; persiste contexto y resultados intermedios; la "source of truth" del lifecycle de un request. |
| **Prompts** | Instrucciones en lenguaje natural que definen la función del LLM en un node; **modulan** el modelo general-purpose en un worker especializado. |
| **Tools** | Interfaces ejecutables para interactuar con el mundo externo; bridgean el gap entre el training data estático y la info real-time dinámica = los Hayhooks services. |
| **Nodes** | Unidades discretas de ejecución donde se aplica lógica; encapsulan un paso del proceso. |
| **Graph** | La orchestration layer que define routing logic y dependencias entre nodes; reemplaza scripts lineales con una state machine que puede loop/branch/react dinámicamente. |

> [!note] Assets **compartidos** entre las 3 arquitecturas: los **tools** (las microservices Yelp Navigator del [[08 - Hands-On Projects|Cap 8]]), los **prompts** (supervisor/summary/clarification nodes) y la **LLM config** (un único LLM). Los nodes asociados a tools son los **worker nodes**.

### Version 1: The sequential chain

![[10-fig-10.1-yelp-navigator-v1.png]]
*Figure 10.1 – Yelp Navigator version 1*

V1 es el **shallow agent pattern** que dominó el early landscape por su simplicidad: un script que corre un prompt, parsea el output, llama un tool y retorna el resultado. Se construye sobre la asunción del **happy path** (el usuario pregunta claro, la API está disponible, el modelo parsea bien, la respuesta se encuentra al primer intento).

El flujo de nodes:

- El **clarification node** toma la query y determina si tiene la info para triggerear el search node (necesita **location + business category**).
- El **search node** usa el `business_search` Hayhooks microservice.
- Según el detail level, mueve a summarization o pide el **details node** (`business_details` microservice); se replica con el `business_sentiment` microservice.
- Al llegar al **summary node**, pasa al **supervisor node** que revisa el reporte final y aprueba, o lo manda de vuelta a los worker nodes si está incompleto. El conditional routing son `if-else` statements en cada worker node.

> [!warning] **Brittleness extrema.** Si el search devuelve resultados irrelevantes por una query vaga, el sentiment step **procede igual** (analizando sentiment de businesses irrelevantes) y el summarization step **alucina** una respuesta coherente de data inconexa.

> [!note] La causa raíz está en su **state definition**: TODA la data (queries, intermediate reasoning, raw JSON outputs) se vuelca en un **único context window creciente** → **context rot**: el modelo se distrae con el ruido de turnos previos e ignora las system instructions.

### Version 2: The router pattern

![[10-fig-10.2-yelp-navigator-v2.png]]
*Figure 10.2 – Yelp Navigator version 2*

V2 abandona la sequential chain por un **[[Supervisor-worker pattern|supervisor pattern]]** con un **clarification step explícito**, desacoplando el entendimiento de la intención del usuario de la ejecución de la tarea. NO asume el happy path: maneja la bifurcación (¿el usuario está chateando o buscando?).

- El **clarification node** es la **primera línea de defensa** (gatekeeper): analiza la query y rutea a un **general chat node** si es conversacional, o promueve al **supervisor node** si requiere data retrieval.
- El **state evolucionó**: V1 volcaba todo en una lista creciente de messages; V2 introduce **structured fields** para data limpia que los tools usan, separando el search context de la raw conversation history; crea state objects separados para clarifier, supervisor y worker nodes.
- Una vez en el supervisor, el sistema ya confirmó que se busca y pobló search query + search location; el supervisor actúa como **router** decidiendo qué worker node (search/details/sentiment) llamar, en vez de correrlos en secuencia forzada.

> [!note] **Diferencia clave de grafo**: en V1 los agent nodes leían el state y decidían el next node; en V2 la decisión pertenece **SOLO al supervisor**. V2 introduce la clase **`Command`** con sus parámetros `goto` y `update` (routing logic embebida en los nodes, decisión dinámica pero determinista).

El código del clarification node:

```python
def clarify_intent_node(
    state: AgentState, config: RunnableConfig
) -> Command[Literal["supervisor", "general_chat", END]]:
…
    if decision.need_clarification and conf.allow_clarification:
        return Command(
            goto=END, update={"messages": [AIMessage(
                content=decision.clarification_question)]}
        )
    if decision.intent == "general_chat":
       return Command(goto="general_chat")
    else:
        # More behavior
```

Las tres ramas: si necesita más info → `END` (el runtime permite al usuario dar más); si la query no se relaciona → general chat; si tiene la info → supervisor → worker node. V2 tiene utilities (`supervisor_utils.py`) con **reglas estrictas** para que el supervisor decida qué hacer si un tool reporta failure (vs V1, donde el supervisor approval node era final y dependía solo de prompts).

> [!tip] Al dejar al supervisor decidir según el state, **V2 elimina la hallucination from disjointed data de V1**: si el search no devuelve resultados, el supervisor puede halt o pedir una nueva query, en vez de forzar sentiment sobre data inexistente.

> [!warning] PERO V2 asume un **benign user y un mundo perfecto** (high-trust assumption): que no habrá prompt injection, que no se leakea **PII (Personally Identifiable Information)** en logs, que las APIs externas nunca hacen timeout/fail. Sus debilidades: si el search falla puede alucinar una razón; si el usuario intenta un override, el clarification lo procesa ciegamente; y le falta **memoria** (si el usuario pide un servicio+location, recibe respuesta y pregunta más sobre un business específico, el agente triggerea una búsqueda entera nueva y "olvida" lo ya obtenido). Para ir de prototipo a producción → **guardrails, resiliency, memory**.

### Version 3: The resilient supervisor

![[10-fig-10.3-yelp-navigator-v3.png]]
*Figure 10.3 – Yelp Navigator version 3*

V3 envuelve la lógica en **capas de protección**: un **defensive perimeter** antes de que el agente piense, y un **safety net** debajo de la ejecución de tools.

- El **entry point** ya NO es el clarify intent node, sino un **deterministic input guardrails node** (`guardrails.py`): NO usa LLM necesariamente, sino lógica rápida y rigurosa —p. ej. **RegEx**— para **sanitizar inputs antes de gastar tokens o arriesgar el sistema**, escaneando PII e injection. Se encapsula como un node nuevo en el grafo.
- La **estabilidad del mundo externo** se define en el grafo con **`RetryPolicy`** (a nivel **infraestructura**, NO en el prompt). En V2 la network flakiness puede crashear; en V3 no:

```python
# Tool Nodes with Retry Policies
retry_policy = RetryPolicy(
    max_attempts=3,
    initial_interval=1.0,
    backoff_factor=2.0,
    max_interval=10.0
)
workflow.add_node(
    "search_tool", search_tool_node, retry_policy=retry_policy)
```

El state trackea retries y consecutive failures:

```python
retry_counts: Dict[str, int] = {}
consecutive_failures: Dict[str, int] = {}
```

Cuando el tool falla 3 veces consecutivas, reporta al supervisor, que según un flag de su state termina el proceso:

```python
        if check_failures and decision.should_finalize_early:
            return "finalize", None, None
```

> [!note] El retry policy aplicado a cada worker node habilita **[[Circuit breaker|circuit breaker]] logic**: si un tool falla repetidamente pese a los retries, el supervisor **degrada gracefully** a un mensaje predeterminado en vez de spiralear en un error loop o alucinar.

La **memoria** llega vía checkpointing implícito en el grafo:

```python
from langgraph.checkpoint.memory import MemorySaver
checkpointer = MemorySaver()
```

En LangGraph Studio hay persistencia multi-run; local, se usa `MemorySaver` (`checkpointer`) → el agente "recuerda" búsquedas previas como contexto, vía in-memory/SQL/PostgreSQL (scripts de in-memory persistence y SQLite persistence). La clave de la in-memory persistence es el **user thread**:

```python
config = {"configurable": {"thread_id": "user_session_123"}}
```

> [!tip] El **thread ID** hace que el agente recuerde respuestas previas como contexto adicional. Al manejar explícitamente el **unhappy path** (malicious inputs, broken tools), V3 es **production-grade** con paths deterministas y observables. La evolución V1→V3 espeja el shift de scripting a engineering: V3 es *"un software system que usa LLMs"*, no *"un LLM wrapper"*.

## The quantitative argument

El libro mide empíricamente la eficiencia con helper scripts y mock tools (`measure_token_usage.py`, `test_examples.sh`, reporte `test4_comprehensive.md`), comparando V1 (los nodes deciden Y ejecutan) vs V2 context-aware (un supervisor decide, los agent nodes solo ejecutan).

### Tabla 10.1 — Token usage across V1 and V2 for different queries

| Query | Detail level | V1 tokens | V2 tokens | Reduction (tokens) | Reduction (%) |
|---|---|---|---|---|---|
| Italian restaurants in Boston, MA | General | 1646 | 1105 | 541 | 32.9% |
| Coffee shops in San Francisco, CA | General | 1656 | 1112 | 544 | 32.9% |
| Pizza places in Chicago, IL | General | 1646 | 1105 | 541 | 32.9% |
| Italian restaurants in Boston, MA | Detailed | 3693 | 1484 | 2209 | 59.8% |
| Mexican restaurants in Austin, TX | Detailed | 3697 | 1488 | 2209 | 59.8% |
| Sushi restaurants in New York, NY | Reviews | 5890 | 1495 | 4395 | 74.6% |
| Vegan restaurants in Portland, OR | Reviews | 5876 | 1487 | 4389 | 74.7% |
| Steakhouses in Dallas, TX | Reviews | 5884 | 1491 | 4393 | 74.7% |

*Table 10.1 – Token usage across V1 and V2 for different queries*

> [!tip] En el escenario *reviews*, V1 **colapsa bajo su propio peso** (~6,000 tokens en una interacción); V2 logra un **74% de reducción**. La diferencia es el manejo de data en el state: **V1 lee TODA la data para decidir; V2 solo chequea si la data existe**.

El mecanismo concreto: en V1 el supervisor approval node incluye los agent outputs en su prompt (re-lee el JSON entero de businesses **cada vez**); en V2 el supervisor acepta **Boolean flags** (`True`/`False`) según si search/details/sentiment devolvieron data. V1 trata los messages como un **scratchpad** (lleno de intermediate data); V2 mantiene la data oculta. La técnica clave es la **summarization**, visible en el state field `pipeline_data` (pasado entre worker nodes pero **nunca renderizado en el supervisor state** → context window limpio).

> [!warning] El context bloat de V1 también se extiende al **tool layer**: los sentiment/detail microservices son naive y toman el general search pipeline results **entero** como input (mejor habría sido ser selectivo). Esto se dejó **intencionalmente sin resolver** para mostrar que el **context engineering no se limita a la orchestration layer, sino que está profundamente atado a las decisiones de la tool layer**. Esto estableció V2 como superior y motivó productionizar V2→V3.

## A case study in microservice failure

El driver primario de la tesis tool-vs-orchestration es la **integrity**: cómo se comporta un agente cuando los componentes fallan. ¿Crashea? ¿Alucina? ¿Degrada gracefully? El experimento apaga **intencionalmente** las Hayhooks microservices y testea tres modelos open-weight en V1/V2/V3 en cada nivel (general/details/sentiment).

- **Modelos**: **OpenAI GPT-OSS 20B, [[DeepSeek-R1]], Qwen 3**.
- **Tres preguntas**: `best pizza places in Chicago`, `best pizza places in Chicago and what reviewers said`, `best pizza places in Chicago and website information`.
- **Script**: `stress_test_architectures.py`, con resultados a temperature 1 y temperature 0.

### Tabla 10.2 — Open weight LLMs' success and failure rates when faced with failing microservices

| Model | Version | Q1 (Basic) | Q2 (+ Reviews) | Q3 (+ Website) |
|---|---|---|---|---|
| gpt-oss:20b | V1 | ✓ 103 s | ✓ 110.6 s | ✖ Recursion limit |
| gpt-oss:20b | V2 | ✖ Recursion limit | ✖ Recursion limit | ✖ Recursion limit |
| gpt-oss:20b | V3 | ✓ 15.7 s | ✓ 14.6 s | ✖ 15.7 s |
| deepseek-r1 | V1 | ✓ 110.6 s | ✓ 81.2 s | ✖ Timeout (>120 s) |
| deepseek-r1 | V2 | ✓ 166.6 s | ✓ 179.1 s | ✓ 184.8 s |
| deepseek-r1 | V3 | ✓ 86.4 s | ✓ 43.9 s | ✓ 37.2 s |
| qwen3 | V1 | ✖ Timeout (>120 s) | ✓ 92.0 s | ✓ 86.4 s |
| qwen3 | V2 | ✖ Recursion Limit | ✓ 186.6 s | ✓ 178.7 s |
| qwen3 | V3 | ✓ 20.3 s | ✓ 34.6 s | ✓ 24.1 s |

*Table 10.2 – Open weight LLMs' success and failure rates when faced with failing microservices*

Hay tres patrones de resultado: el agente **"succeeds"**, falla por **recursion limit**, o falla por **timeout** (impuesto por el script). DeepSeek-R1 resolvió consistentemente en V1 y V2; GPT-OSS y Qwen3 dieron resultados mixtos.

### Tabla 10.3 — Agent behavior seen via the nodes

| Pattern name | Node sequence | Engineering insight |
|---|---|---|
| Recursion trap | Super → Search... | Recursion limit: Infinite loop until crash |
| Deep retry | Super → Search (x5) → Sum → End | DeepSeek/Qwen3: Brute-force retries until "success." |
| Safe exit | Super → Search → Super → End | The V3 Fix: Detects failure immediately and quits. |

*Table 10.3 – Agent behavior seen via the nodes*

Los tres patrones de comportamiento:

- **Recursion trap** (V1 y V2) — la query se "atasca" en un worker node y triggerea el **recursion limit de LangChain**.
- **Deep retry** (V1 y V2) — loop de ida y vuelta entre worker nodes y supervisor (en V1 el supervisor tiene un approval limit, en V2 un timeout). Con **temperature 0**, GPT-OSS reportó que no se encontró data; con **temperature 1, alucinó businesses con phone numbers, emails y ratings**. Con Qwen3 y DeepSeek-R1, el summary node **alucinó una respuesta final** con businesses/reviews/phone numbers **sin importar la temperature**.
- **Safe exit** (V3) — todos los modelos salieron gracefully con: `I apologize, but I'm unable to complete your request due to service unavailability. The Yelp API service is currently unavailable or has rate limits in effect. Please try again later`.

### Tabla 10.4 — Execution path of V3 during microservice failure

| Step | Node | Action | Status |
|---|---|---|---|
| 1 | Guardrails node | Scan for PII or injection | Pass |
| 2 | Clarify | Determine user intent | Pass |
| 3 | Supervisor | Plan: Call search tool | Pass |
| 4 | Search node | Execute HTTP request | FAIL (503/429) |
| 5 | Supervisor | Analyze failure signal | Circuit break |

*Table 10.4 – Execution path of V3 during microservice failure*

> [!note] El momento crítico es el **step 4**: el search node (microservice wrapper) **cazó el error**; en vez de crashear el proceso Python o pasar raw HTML al supervisor, retornó un **structured error signal**; intentó 3 veces y backed off, y retornó al supervisor que triggerea el safety exit.

Lo más fascinante es **qué nodes alucinaron**. En V1/V2, los modelos en deep retry entraron en loop (el search node le decía al supervisor que falló; el supervisor insistía hasta el timeout). El **summary node** recibió la señal `"No data was found"` y aun así generó una respuesta completa con businesses listados, reviews, phone numbers, websites y de qué es conocido el "restaurant".

> [!warning] Insight honesto del autor: esto fue **engineered para frustrar al LLM**. Los nodes constreñidos por la **determinism de comandos Python o pipelines Haystack NO alucinaron**; el node que **caved** fue el constreñido por la **naturaleza subjetiva del prompt**. Cuando el modelo recibió "no data found" pero el prompt le instruía que DEBE responder, su reinforcement learning de ser **helpful** priorizó sobre reportar que no había nada. *"How you design your system matters."*

> [!warning] Lo más aterrador de V1: **todas las instancias fueron aprobadas por el supervisor salvo que ocurriera un timeout**. Si tu sistema falla con una arquitectura V1-like que NO trackea el tool layer por separado, no te enterás salvo por timeout.

Casos reales de production failures por falta de guardrails:

- **NYC legal brief (2023)** — un abogado usó ChatGPT para precedentes; presionado, el modelo **inventó casos judiciales falsos**.
- **Air Canada chatbot (2024)** — alucinó una **refund policy inexistente** por ser presionado a ser helpful sin chequear la DB real; la aerolínea fue **legalmente forzada a pagar** el refund.
- **Multi-agent infinite loop (2025)** — 4 AI agents para market data; 2 quedaron en un loop de conversación autónoma por **11 días**, generando **$47,000** en API costs antes del shutdown manual.

> [!tip] El éxito de V3 es un testimonio de que **orchestration outweighs intelligence**: en vez de buscar un modelo "smart" que intuya que Lucali no está en Chicago, la perspectiva de ingeniería prioriza un **circuit breaker** que impide al modelo intentar una respuesta cuando la DB está caída. V3 dejó de depender de la "honestidad" probabilística del LLM e impuso **deterministic state checks**. Tratar al LLM como un componente no fiable dentro de una arquitectura fiable = aplicar **[[Site Reliability Engineering (SRE)]] a la IA**.

## Context engineering in the Yelp Navigator V3

La evolución V1→V3 ES el shift de prompt engineering a **[[Context Engineering|context engineering]]**. Las cuatro estrategias transformaron un script propenso a hallucinations en un sistema production-grade.

> [!note] **Write** — el contexto debe ser explícito y estructurado, no escondido en un chat log. En V3, vía la clase `AgentState`: a diferencia de V1 (que volcaba todo en una historia lineal), V3 "escribe" facts críticos (search query, total error count) en **state fields dedicados**.
> **Engineering gain:** habilitó la "self-awareness" — trackeando retry counts en el state, los modelos V3 detectaban sus propios fallos y triggereaban un circuit breaker, evitando los "recursion traps" que crashearon V2.

> [!note] **Select** — recuperar solo la data necesaria para la tarea inmediata. En V3 el supervisor node actúa de **selector** usando `Command` para rutear a tools específicos, en vez de correr una cadena monolítica.
> **Engineering gain:** drove un **27.7% de reducción de tokens** para general queries (Tabla 10.1); previno que el modelo se distrajera con outputs irrelevantes, resolviendo la brittleness de V1.

> [!note] **Compress** — maximizar la utilidad del context window destilando payloads masivos. V3 lo implementa vía el **summary node** (procesa raw API JSON en natural language conciso antes de appendearlo).
> **Engineering gain:** lo más visible en queries "Review" complejas — V1 colapsó bajo **5,659 tokens** de raw data; V3 logró un **75.1% de reducción** (a ~1,400 tokens). La eficiencia no es solo costo: es **preservar el cognitive bandwidth** del modelo.

> [!note] **Isolate** — particionar el contexto para prevenir cross-contamination entre tasks. En V3, vía el **clarify intent node** que triggerea un "hard reset" del pipeline data al detectar un topic switch.
> **Engineering gain:** previene el context bloat de V1; cada búsqueda nueva empieza con un **clean cognitive slate**.

> [!tip] **La validación última**: en los failure mode experiments, los modelos "smarter" en V2 (DeepSeek-R1, Qwen3) **alucinaron** businesses inexistentes con websites y phone numbers; V3 (fortificada por estos principios) triggereó un safety exit el **100% de las veces**. Esto prueba que la fiabilidad NO es propiedad del modelo, sino del **contexto en el que opera**. Tratar al LLM como un componente no fiable dentro de un sistema fiable y context-engineered bridgea el gap entre un stochastic demo y un deterministic product.

> [!note] Detalle notable: **no se gastó un centavo** en el stress test — todos los modelos corren en una laptop gracias a **Ollama** y a las decisiones arquitectónicas (limitación: los modelos usados soportaban thinking, tool calling y structured parsing).

## The sovereign stack

Las mejoras de V3 (isolation, state resilience, specialized prompting) habilitan el shift a **[[Sovereign agent|sovereign agents]]**.

> [!note] Un **sovereign agent** es un AI system que corre **enteramente dentro de la infraestructura del usuario**, independiente de API providers centralizados (OpenAI, Anthropic), asegurando **data privacy y autonomía operativa**.

El argumento económico se apoya en el **[[TCO (Total Cost of Ownership)|TCO]]** (introducido en el [[02 - Diving Deep into Large Language Models|Cap 2]]), dominado por los **inference costs**:

- Los modelos API tienen bajo upfront pero **costos variables que escalan linealmente** con el uso.
- Para un multi-agent system con supervisor, un único user request puede triggerear **50 internal reasoning steps** (loops); pagando por token a GPT-4, ese cost structure es **prohibitivo**.
- Correr un sovereign agent en hardware local (consumer GPUs o edge devices) cambia la economía: el **marginal cost del loop número 50 es efectivamente cero** (ignorando electricidad) → V3 económicamente viable para apps high-volume.

> [!warning] Para apps enterprise con **data sensible** (healthcare, finance), mandar data a una API pública es un **non-starter**. La data sovereignty no es un lujo: es un requisito.

La arquitectura sovereign agent asegura las tres capas en local:

| Capa | Implementación local |
|---|---|
| **Data ingestion** | Pipelines [[Haystack 2.0|Haystack]] |
| **Inference** | Ollama o vLLM + NVIDIA y modelos soportados |
| **State management** | [[LangGraph]] |

> [!tip] Esta **data sovereignty completa** es el deliverable último del patrón tool-vs-orchestration combinado con open-weight models.

## Summary (cierre del libro)

El libro transformó **[[RAG]]** de varios formatos (text, multimodal, vector DB, agents, persistencia) en una arquitectura cohesiva production-grade, y examinó formas de testearlo (knowledge graphs, unit testing, token cost, integrity testing). Las cuatro estrategias write/select/compress/isolate dan ganancias medibles (token reduction, context rot reduction, system integrity). Apagando los tools en un entorno controlado se demostró que **la fiabilidad no es propiedad de la inteligencia del modelo, sino resultado de la arquitectura del sistema**: un sistema bien arquitecturado (V3) reduce el token usage **>74%** vs una implementación naive. Desacoplando el *doing* (Haystack) del *thinking* (LangGraph) + open-weight models → **sovereign agents** que corren localmente, protegen privacy y fallan gracefully en vez de alucinar cuando las APIs caen.

> [!tip] **Cuatro principios arquitectónicos** para llevar a futuros proyectos:
> - **The tool versus orchestration pattern** — no le pidas al cerebro hacer el heavy lifting: Haystack para data pipelines deterministas (tool layer), LangGraph para stateful reasoning y control flow (orchestration layer).
> - **Data-centricity over prompting** — la fiabilidad viene de **mejor data, no solo mejores prompts**; usar [[Ragas]] + synthetic data generation para testear rigurosamente antes de producción.
> - **The sovereign stack** — la era de depender solo de APIs centralizadas pagas está terminando; open-weight models + arquitecturas robustas = agentes económicamente sostenibles y bajo tu control.
> - **Engineering integrity** — tratar a los LLMs como **Stochastic Processing Units (SPUs)**: componentes potentes pero no fiables que deben envolverse en código determinista, guardrails y retry policies; priorizar nodes especializados con tools de alta calidad + un orquestador que rutea; y asegurar que el final node esté bound por **instrucciones deterministas** (como la supervisor decision function).

## Citas

> "We have arrived at a critical inflection point where the probabilistic nature of LLMs clashes with the deterministic requirements of business operations."
> "If Haystack represents the body of the agent... then LangGraph represents the brain."
> "The node that caved was the node constrained by the subjective nature of the prompt."
> "The decisive success of V3 serves as a testament to the principle that orchestration outweighs intelligence."
> "This proves that reliability is not a property of the model, but of the context in which the model operates."
> "Treat LLMs as Stochastic Processing Units (SPUs)."
> "This moves us from building impressive demos to deploying trustworthy enterprise software."

## Para aplicar

- **Aplicar el patrón tool-vs-orchestration** — [[Haystack 2.0|Haystack]] como tool layer determinista (pipelines/[[Hayhooks]] como microservicios, el agente hace `HTTP POST` y los trata como deterministic black boxes) + [[LangGraph]] como orchestration layer stateful.
- **Diseñar las 3 capas de defensa de V3** — (1) un **guardrails node determinista** como entry point (RegEx, scan de PII/injection antes de gastar tokens); (2) una **`RetryPolicy`** a nivel grafo en los tool nodes (`max_attempts=3, initial_interval=1.0, backoff_factor=2.0, max_interval=10.0`); (3) un **[[Circuit breaker|circuit breaker]]** (consecutive failures → safe exit); y (4) **persistence/memory** con `MemorySaver` + `thread_id`.
- **Reducir tokens en el supervisor** — usar **Boolean flags** en el supervisor state (NO re-leer el JSON entero) + un `pipeline_data` summarizado **oculto del supervisor**.
- **Aplicar las 4 estrategias de context engineering** — **write** = state fields explícitos (`AgentState`); **select** = `Command` routing a tools específicos; **compress** = summary node que destila JSON a NL; **isolate** = hard reset del pipeline data en topic switch.
- **Construir un sovereign stack** — Haystack ingestion local + Ollama/vLLM inference local + LangGraph state local, con open-weight models (**GPT-OSS / [[DeepSeek-R1]] / Qwen 3**) para **data privacy** y un **[[TCO (Total Cost of Ownership)|TCO]]** viable (marginal cost del loop ≈ 0).
- **Tratar al LLM como un SPU** — envolverlo en código determinista; asegurar que el final/supervisor node esté bound por una **decision function determinista**, NO solo por prompts.
- **Stress-testear apagando los tools** — verificar **graceful degradation** (safe exit, no hallucination) antes de producción; recordá que una arquitectura V1-like que no trackea el tool layer aparenta éxito salvo timeout.

## Conexiones

- [[_Building Natural Language and LLM Pipelines|Building NL & LLM Pipelines]] — el MOC del libro.
- [[09 - Future Trends and Beyond]] — capítulo anterior; aporta los protocolos [[Model Context Protocol (MCP)]]/[[A2A]] y la memoria evolutiva ACE que este Epílogo **aterriza** en la arquitectura concreta de V3. (Es el último capítulo: no hay capítulo siguiente.)
- [[08 - Hands-On Projects]] — el **Yelp Navigator** y las microservices Hayhooks que aquí **evolucionan** a V1/V2/V3.
- [[01 - Introduction to Natural Language Processing Pipelines]] — la **agentic reliability crisis** que el Epílogo **cierra** con "SRE for AI".
- [[02 - Diving Deep into Large Language Models]] — las 4 estrategias de [[Context Engineering|context engineering]], el [[TCO (Total Cost of Ownership)|TCO]] y [[DeepSeek-R1]]/[[GRPO]].
- [[05 - Haystack Pipeline Development with Custom Components]] · [[06 - Building Reproducible and Production-Ready RAG Systems]] — [[Ragas]], synthetic data y la **data-centricity** del cierre.
- [[07 - Deploying Haystack-Based Applications]] — los [[Hayhooks]] microservices que aquí actúan como tool layer.
- [[Haystack 2.0]] · [[Hayhooks]] · [[LangGraph]] · [[RAG]] · [[Context Engineering]] · [[TCO (Total Cost of Ownership)]] · [[Sovereign agent]] · [[Site Reliability Engineering (SRE)]] · [[Supervisor-worker pattern]] · [[Circuit breaker]] · [[DeepSeek-R1]] · [[GRPO]] · [[Ragas]] — el stack arquitectónico del Epílogo.
- **Stochastic Processing Unit (SPU)** · **shallow agent** · **deep agent** · **RetryPolicy** · **MemorySaver / checkpointer** · **Command (LangGraph)** · **guardrails node** · **recursion trap** · **deep retry** · **safe exit** · **GPT-OSS** · **Qwen 3** · **Ollama** · **vLLM** · **happy path / unhappy path** — conceptos clave del capítulo; candidatos a nota propia.
