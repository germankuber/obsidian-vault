---
title: 02 - Diving Deep into Large Language Models
libro: Building Natural Language and LLM Pipelines
autor: deepset / Packt
capitulo: 2
created: 2026-06-12
tags:
  - libros/building-natural-language-and-llm-pipelines
  - type/case-study
  - status/permanent
aliases:
  - Diving Deep into Large Language Models
  - Cap 2 - LLMs
updated: 2026-06-12
---

# 02 - Diving Deep into Large Language Models

> [!info] Capítulo 2 · *Building Natural Language and LLM Pipelines* — Packt (ISBN 9781835467992)
> Traza la evolución del LLM monolítico de 2023 al stack ágentico especializado de 2025: la [[Transformer|arquitectura transformer]], el toolkit de 2023 ([[RAG]], prompting, fine-tuning), la gran especialización en **SLMs** y **RLMs** (con el [[DeepSeek-R1|DeepSeek Moment]] y [[GRPO]]), el salto de prompt a **[[Context Engineering|context engineering]]**, los frameworks de 2025 ([[LangGraph]] + [[Haystack 2.0]]) y la economía del deployment — la **inference** (no el training) es el costo dominante. Justifica la arquitectura híbrida del libro: **LangGraph como orchestration layer, Haystack como tool layer**. Navegá: [[_Building Natural Language and LLM Pipelines|Building NL & LLM Pipelines]] · anterior [[01 - Introduction to Natural Language Processing Pipelines]] · siguiente [[03 - Introduction to Haystack by deepset]].

## Resumen

El capítulo profundiza en los LLMs trazando su evolución desde la baseline de 2023 hasta las arquitecturas ágenticas especializadas de 2025, en cuatro ejes: **overview de LLMs**, **interacting with language models**, el rol de los **vector stores** (data storage/retrieval/memory) y la **economía del deployment** (el costo de inference). Parte de las cuatro categorías de uso (**understanding, generating, retrieving, interacting**), repasa la [[Transformer|transformer architecture]] (*Attention Is All You Need*, Vaswani et al. 2017, rev. 2023) con sus 7 componentes, y las tres familias dominantes de 2023 — **GPT** (auto-regresivo), **[[BERT]]** (bidireccional) y **BART** (combinación) — con sus tamaños (GPT 110M → GPT-4 1.7T). Identifica los dos desafíos del modelo monolítico (**hallucinations** + costo computacional) y el toolkit de 2023 (**prompting, fine-tuning, [[RAG]]**) más las técnicas cost-aware (**IT, RLHF, PEFT, LoRA, model distillation**).

El giro central: el paradigma "one-size-fits-all" se fracturó en dos clases opuestas — **SLMs** (small language models, <10B params, hyper-efficient, edge/on-device) y **RLMs** (reasoning language models, arquitecturas híbridas con search heuristics tipo MCTS). El **[[DeepSeek-R1|DeepSeek Moment]]** (R1, 20 enero 2025, licencia MIT, training bajo $6M, a la par de OpenAI-o1) democratizó el razonamiento avanzado, apoyado en el algoritmo **[[GRPO]]** (elimina el critic model usando estadísticas de grupo) y en un pipeline de training multi-etapa. Esto cambió el rol del developer: de escribir prompts aislados a orquestar contextos enteros — la disciplina del **[[Context Engineering|context engineering]]** (write, select, compress, isolate), superset del prompt engineering, que combate el **context rot** y la **context pollution**.

Luego justifica la arquitectura del libro — *"not LangGraph or Haystack but LangGraph and Haystack"* ([[LangGraph]] orchestration + [[Haystack 2.0]] tool layer) — analizando la especialización de frameworks en 2025 (LangChain 1.0 sobre LangGraph 1.0; Haystack 2.0 + [[Hayhooks]]). Profundiza en el rol evolucionado del **vector store**: de índice [[RAG]] estático read-only (2023) a capa de memoria ágentica dinámica read-write (2025), con su pipeline de ingestion, el **hybrid search** (BM25 + dense, fusionados con RRF), las capas de memoria y el problema de **memory curation** (consolidation ADD/UPDATE/NO-OP de AgentCore). Termina con la economía: la **inference** (no el training) es el costo dominante, "The Great AI Cost Compression of 2025", el **TCO**, el latency-throughput trade-off, y la decisión private vs hosted (capability, cost, latency, privacy).

## Overview de LLMs

Un **LLM** es una categoría de modelos de deep learning para procesar y producir texto human-like, con arquitecturas como los [[Transformer|transformers]] (Vaswani et al. 2017, 2023) y de cientos de millones a miles de millones de parámetros, entrenados en datasets diversos (libros, websites, artículos). Su tamaño exige recursos computacionales significativos en training e inference.

### Use cases de LLMs

Cuatro categorías principales de aplicación:

- **Understanding (classic NLP)** — analizar reviews, posts de social media o encuestas para sentiment (positive/negative/neutral); clasificación de texto (spam detection, topic classification); y named-entity recognition (identificar personas, organizaciones, lugares).
- **Generating (text generation)** — creación de contenido (marketing, social media) y generación de código desde descripciones en lenguaje natural. **GitHub Copilot** es el ejemplo prima, evolucionado de simple code completion a un sistema ágentico sofisticado — parte del **agentic DevOps** (Microsoft, 2025), donde agentes colaboran con developers en cada etapa del ciclo de software. Otra familia mayor es **Anthropic Claude** (serie 3.5: Sonnet/Opus/Haiku); Claude 3.5 Sonnet marca benchmarks en razonamiento graduate-level (GPQA) y coding (HumanEval), a menudo superando a GPT-4o.
- **Retrieving (grounding en datos)** — mejorar search entendiendo el contexto de queries, Q&A sobre documentación, y extraer info de documentos para poblar DBs estructuradas.
- **Interacting (agentes conversacionales)** — chatbots (ChatGPT), virtual assistants, content moderation bots; soporte al cliente, FAQs, detección/filtrado de contenido inapropiado.

### La baseline de 2023: la arquitectura Transformer

El engine de la revolución LLM es la **transformer architecture** (*Attention Is All You Need*, Vaswani et al. 2017 rev. 2023), que abandonó el procesamiento recurrente secuencial por el **procesamiento paralelizado de tokens**, capturando dependencias globales sin procesar cada posición una a una.

![[02-fig-2.1-transformer-architecture.png]]
*Figure 2.1 – Transformer architecture, inspired by the process outlined in Vaswani et al.*

Sus 7 componentes clave:

1. **Inputs** — el texto se tokeniza, sus tokens reciben IDs únicos, y los IDs se vectorizan con una embedding layer.
2. **Positional encoding** — inyecta la posición relativa o absoluta de cada token (los transformers no entienden el orden por sí solos). **Absolute** (transformer original) o **relative** (Shaw et al., 2018, para tareas donde la posición relativa importa más).
3. **Attention mechanism** — de dos tipos: **self-attention (dot-product)** (pesa la relevancia de cada palabra respecto a todas las del enunciado, cuantificando cuánta atención prestarle) y **multi-head attention** (self-attention en paralelo múltiples veces, cada "head" con distintos pesos, resultados concatenados — permite atender a distintas partes de la secuencia).
4. **Feed-forward neural networks** — red fully-connected por posición, dos transformaciones lineales con ReLU intermedia; refina la comprensión del input en cada posición.
5. **Layered normalization** — estabiliza y acelera el aprendizaje; se aplica tras cada bloque attention y feed-forward.
6. **Output layer** — capa lineal → logits, softmax → probabilidades del próximo token; distribución sobre vocabulario (generación) o sobre clases (clasificación).
7. **Post-processing** — convierte los token IDs del vector de vuelta a lenguaje humano.

Este diseño dio lugar a **tres familias dominantes en 2023**:

- **GPT** (generative pre-trained transformer, OpenAI) — decoder, **auto-regresivo**, predice la siguiente palabra. Tamaños: GPT 110M, GPT-2 1.5B, GPT-3 175B (Radford et al. 2018/2019/2020), GPT-4 1.7T (OpenAI 2023). Bueno para tareas generativas (stories, summaries, código).
- **[[BERT]]** (bidirectional encoder representations from transformers, Google, Devlin et al. 2018) — encoder bidireccional multi-capa con bidirectional masking + next-sentence prediction. Bueno para comprensión del input (Q&A).
- **BART** (bidirectional and auto-regressive transformers, Facebook AI, Lewis et al. 2019) — combina ambos: BERT es bueno para clasificación pero malo para generación; GPT bueno para generación pero malo para tareas que requieren la secuencia completa.

#### Tabla 2.1 — Popular transformer models

| Transformer type | Architecture | Model-like | Focus | Example |
|---|---|---|---|---|
| Auto-regressive | Decoder | GPT-like | Generative tasks | Chatbot |
| Auto-encoding | Bidirectional encoder | BERT-like | Understanding of the input | Question-answering |
| Sequence-to-sequence | Encoder-decoder | BART-like | Generative tasks that require an input | Language translation |

*Table 2.1 – Popular transformer models*

### El toolkit de interacción de 2023: prompting, fine-tuning y RAG

Los dos desafíos primarios de los modelos monolíticos de 2023 eran las **hallucinations** (contenido plausible pero factualmente incorrecto) y el inmenso costo computacional de training, fine-tuning e inference. Surgieron tácticas mitigadoras:

- **Prompting** — proveer un input o instrucción específica (el "prompt") para guiar la respuesta.
- **Fine-tuning** — entrenar un LLM pre-entrenado en un dataset domain-specific de pares instruction-output.
- **[[RAG]]** (Lewis et al. 2021) — combina un modelo paramétrico pre-entrenado con un índice de vectores densos no-paramétrico de documentos externos.

Técnicas de fine-tuning:

- **Instruction Tuning (IT)** (Zhang et al. 2023) — entrena con pares instruction-output (instrucción humana + respuesta deseada).
- **RLHF** (Reinforcement Learning from Human Feedback, Ouyang et al. 2022, aplicado sobre GPT-3) — tres pasos: demonstration data para una supervised policy → comparison data para un reward model → optimización de la policy contra el reward model con RL.

Técnicas cost-aware:

- **PEFT** (Parameter-Efficient Fine-Tuning, Lialin et al. 2023) — adapta un subconjunto de parámetros en vez de todos (en LLMs los params son miles de millones; fine-tunear cada uno es caro e innecesario).
- **LoRA** (Low-Rank Adaptation, Hu et al. 2021) — **congela** las capas pre-existentes del transformer y representa los weight updates con **low-rank decomposition** (matrices más chicas entrenables); acelera el fine-tuning con menos memoria; el resultado final combina pesos originales + adaptados.
- **Model distillation** — un LLM "teacher" entrena un "student model" más chico para que se comporte similar.
- **[[RAG]]** en su paper original (Lewis et al. 2021): memoria paramétrica = un transformer seq2seq pre-entrenado (específicamente **BART**); memoria no-paramétrica = un índice de vectores densos de **Wikipedia** vía neural retriever. Tratando los docs recuperados como variables latentes, se entrena **end-to-end** combinando generación "closed-book" con retrieval "open-book". Reduce hallucinations y permite actualizar conocimiento reemplazando el índice. Los pipelines de 2025 usan modelos como `GPT-4o-mini`.

> [!note] **De "RAG o fine-tuning" a "RAG Y modelos especializados".** En 2023, RAG/PEFT/LoRA eran opciones peer-level competitivas (criterios: **RAG** para datos up-to-date/factuales/propietarios y reducir hallucinations; **PEFT/LoRA** para adaptar a dominio/estilo/formato sin tunear todos los params). En 2025, RAG evolucionó de táctica mitigadora a **patrón arquitectónico fundacional** de confiabilidad empresarial; el fine-tuning y los nuevos métodos RL son parte del propio pipeline de creación del modelo. El stack moderno no es "RAG *o* fine-tuning" sino sistemas que usan RAG *y* modelos especializados.

### La evolución 2024-2025: SLMs y RLMs

El paradigma one-size-fits-all se fracturó en un mercado de **gran especialización**, bifurcado en dos direcciones opuestas:

- **SLMs (small language models)** — la tendencia "smaller, faster"; típicamente **<10B params** con arquitecturas streamlined. Ejemplos: Microsoft **Phi-3-mini** (3.8B), Tsinghua **MiniCPM-2.4B**, Apple **OpenELM**. Fuertes en classification/extraction/lightweight reasoning. Su valor estratégico: **cost efficiency** (usar un modelo masivo para tareas de alto volumen es ineficiente) y **edge AI / on-device deployment** (corren localmente en hardware resource-constrained: smart cameras, in-vehicle assistants, wearable health monitors → real-time, zero-latency y privacidad, sin enviar datos a la nube).
- **RLMs (reasoning language models)** — la tendencia "smarter, deeper"; **no son LLMs escalados sino una arquitectura híbrida nueva** (Besta et al. 2025) diseñada para razonamiento multi-paso deliberado, no solo next-token prediction. Ejemplos 2025: OpenAI **o3** y **o4-mini**, y el open-source **[[DeepSeek-R1]]**.

> [!note] **Blueprint de un RLM (Besta et al. 2025, 3 componentes).** (1) **The LLM (as knowledge base)** — los pesos de un LLM tradicional dan world knowledge y fluidez; (2) **Reinforcement learning (as policy/value model)** — un framework RL guía la generación optimizando una estrategia/outcome a largo plazo en vez de next-token prediction; (3) **Search Heuristics (as explorer)** — algoritmos como **MCTS** (Monte Carlo Tree Search) o **Beam Search** exploran un árbol de pensamientos / grafo de razonamiento, generando múltiples paths, evaluándolos y haciendo backtrack de los poco prometedores.

#### El DeepSeek Moment y GRPO

El evento más significativo del open-source 2025 fue el **DeepSeek Moment**: el **20 de enero de 2025** la empresa china DeepSeek lanzó su modelo **R1**, con significancia doble:

- **Performance** — razonamiento a la par de los mejores modelos cerrados, incluyendo OpenAI-o1.
- **Accessibility** — licencia **MIT** permisiva, con training reportado radicalmente cost-efficient (**bajo $6 millones**).

Esto **democratizó el razonamiento avanzado**, probando que no era dominio exclusivo de corporaciones billonarias. El éxito no fue por scaling simple sino un breakthrough algorítmico y arquitectónico. Se construyó sobre el base model **DeepSeek-V3** (diciembre 2024); la innovación real estuvo en el **training pipeline**.

> [!note] **[[GRPO]] (Group Relative Policy Optimization)** (Deepseek et al. 2025; Shao et al. 2024) — el breakthrough algorítmico core. En un RL típico para LLMs, un **critic model** (reward model neural) estima un baseline para calcular el "advantage" de un output; como ese critic suele igualar al policy model en tamaño, **duplica** memoria y compute. GRPO **elimina el critic separado**: para cada pregunta muestrea un **grupo** de outputs, y calcula el advantage de cada respuesta relativo a la **media y desviación estándar** de los rewards del grupo. Mismo signal de optimización que el RL tradicional, con mucha menos complejidad computacional y de ingeniería.

El **DeepSeek-R1** final fue producto de un pipeline de training multi-etapa:

1. **Phase 1, Pure RL** — entrenaron **DeepSeek-R1-Zero** solo con RL (GRPO) sobre el base model. Éxito, pero con poca legibilidad y language mixing.
2. **Phase 2, Multi-stage** — para el R1 pulido: (a) **SFT cold-start** sobre un set chico de ejemplos chain-of-thought largos de alta calidad (scaffold inicial de razonamiento estructurado); (b) **GRPO** para empujar más allá de la imitación de patrones hacia problem-solving deliberado stepwise; (c) **rejection sampling** para cosechar los mejores outputs del modelo RL-enhanced, ensamblando un dataset sintético más grande y limpio para una segunda ronda de **SFT**; (d) un **RL global final** en todos los escenarios como fase de pulido (alinea profundidad de razonamiento, consistencia de formato y performance).

> [!tip] La lección: el razonamiento avanzado es un desafío **algorítmico y arquitectónico** (resoluble con técnicas RL novedosas + pipelines de training ingenierizados), no meramente un desafío de scaling. Para developers, el razonamiento pasó a ser una herramienta open y accesible.

## Interacting with language models

La evolución 2023→2025 cambió de raíz cómo los developers trabajan con LLMs. Con SLMs y RLMs divergiendo, y agentes operando sobre largas cadenas de decisiones, el **context window** dejó de ser un buffer de input pasivo para volverse una **superficie de ingeniería activa**. El rol del developer se expandió de craftear instrucciones aisladas a orquestar **contextos enteros** (prompts, retrieved knowledge, tool definitions, agent state, intermediate reasoning).

> [!note] **[[Context Engineering|Context engineering]]** es la disciplina de curar, estructurar, comprimir, aislar y gestionar la info que un modelo consume en el tiempo. Beneficios: **enhanced reliability/factuality** (groundear el modelo en datos verificados — defensa primaria contra hallucinations), **mitigación de context rot** (evitar la degradación por inputs largos/ruidosos) y **operational cost efficiency** (técnicas de compress reducen el costo recurrente de inference).

### De prompt a context engineering

- **Prompt Engineering (el craft de 2024)** — el arte de craftear prompts efectivos: instrucciones precisas para obtener una respuesta one-shot deseada. Descrito más exactamente como *"prompt crafting"*; efectivo para tareas simples pero **se rompe** en aplicaciones complejas multi-step.
- **Context Engineering (la disciplina de 2025)** — la disciplina formal nueva, **superset del prompt engineering**. El foco se corre del prompt a *"the delicate art and science of filling the context window with just the right information for the next step"*. Su universo incluye no solo el prompt sino los docs recuperados con RAG, las tool/API definitions, el chat history (memory) y los outputs de pasos ágenticos previos.

> [!warning] **Por qué se rompió el prompting simple.** Un agente en loop genera cada vez más datos, lo que sin disciplina formal lleva a fallos críticos: **context pollution** (una hallucination de un paso contamina los siguientes), **ad hoc string management** (brittle, undebuggable, unscalable), y el **context rot** (Chroma Research, 2025: la performance degrada en contextos largos y ruidosos). Problemas adicionales: **context distraction**, **context confusion** y **context clash** (Kubiya AI, 2025).

### Las 4 estrategias core de context engineering (2025)

- **Write** — almacenar info externamente al context window para retrieval posterior; fundamento de la agent memory. Técnicas: **scratchpads** (memoria de trabajo intermedia para pensamientos/cálculos) y **long-term memories** (preferencias de usuario, facts de conversaciones previas, en una DB externa — a menudo un vector store).
- **Select** — traer dinámicamente solo la info más relevante al context window en el momento que se necesita (estrategia "just in time"). Técnicas: RAG, memory selection (recuperar items del write vector store), dynamic tool selection.
- **Compress** — destilar info grande en representaciones token-efficient para compactar el contexto y evitar context rot. Técnicas: summarization y **tool result clearing** (ej. un agente recibe un JSON blob grande; tras extraer el fact necesario, remueve el JSON crudo del contexto y lo reemplaza por el fact extraído).
- **Isolate** — patrón arquitectónico avanzado: dividir una tarea compleja en compartimentos independientes con context windows limpios y aislados. Técnica clave: **sub-agent architectures** (un supervisor con un goal complejo como deep research spawnea un subagent que corre en su propio context window limpio, hace el deep research y retorna solo un summary destilado — evitando contaminar el contexto primario del supervisor).

## La evolución de los frameworks en 2025

Los frameworks de la era 2023 (LangChain, LlamaIndex, Haystack) se especializaron a lo largo de 2024-2025. El "ad hoc chain" de LangChain fue reemplazado por un **agent engineering stack formal** (LangChain 1.0, LangGraph, LangSmith); deepset pasó de Haystack 1.0 a **2.0**, lanzó deepset Cloud e introdujo capacidades ágenticas. El libro usa dos: **[[LangGraph]] (orchestration layer)** + **[[Haystack 2.0]] (tool layer)** — separación de concerns entre agentic control flow y robust data flow.

### LangChain/LangGraph 1.0: la orchestration layer ágentica

En **octubre 2025** LangChain anunció sus releases v1.0; el pivot clave fue reconstruir **LangChain 1.0** (API high-level) sobre **LangGraph 1.0** (runtime low-level). LangGraph es un runtime graph-based para orquestación de agentes, con un **stateful graph** que soporta explícitamente:

- **Loops** — el patrón fundamental del agente: `Think → act → observe → think`.
- **Conditional branching** — decisiones dinámicas.
- **Durable state and checkpointing** — el estado persiste (pausar/reanudar), crítico para tareas long-running o human-in-the-loop (HITL).

LangChain 1.0 se posiciona como el *"fastest way to build an AI agent"*; su feature definitorio es la abstracción `create_agent`, construida sobre **middleware** — *"a set of hooks that allow you to customize behavior in the agent loop, enabling fine grained control at every step an agent takes"*.

> [!tip] El **middleware es la capa de implementación directa del context engineering**. Hooks como `before_model` y `wrap_tool_call` permiten controlar los tres tipos de contexto: **model context** (dynamic prompts), **tool context** (dynamic tool selection) y **life-cycle context** (summarization, guardrails).

### Haystack 2.0: la data y tool layer robusta

[[Haystack 2.0]] consolidó su posición 2025 como el framework **"pipeline-first"** para pipelines de datos robustos, medibles y auditables, sobre un **directed graph (DG)** explícito con contratos input/output estrictos. Features clave 2025:

- **Enterprise-grade RAG** — ecosistema battle-tested de components para hybrid retrieval (sparse/[[BM25]] + dense/embedding), reranking, y conexión a todas las vector DBs mayores.
- **Measurability and auditing** — pipelines explícitos, transparentes y debuggables; **evaluation nodes built-in** para métricas de calidad RAG, lo que lo hace preferido en compliance e industrias reguladas.
- **Deployment con [[Hayhooks]]** — despliega cualquier pipeline Haystack como REST API o **MCP Server** production-ready con un único comando.

> [!note] Los análisis de mercado 2025 posicionan a **LangGraph como líder en orquestación stateful compleja** y a **Haystack como líder en data pipelines confiables y medibles**. No es conflicto sino un ecosistema maduro — apunta directo a la arquitectura híbrida.

## La arquitectura híbrida: tool layer vs orchestration layer

La arquitectura del libro es **desacoplada y microservice-based**, con separación de concerns: *"This architecture is not LangGraph or Haystack but LangGraph and Haystack."*

### Tabla 2.2 — Framework Specialization in 2025 (LangGraph vs Haystack)

| Feature | LangGraph 1.0 (orchestration layer) | Haystack 2.0 (tool layer) |
|---|---|---|
| **Core focus** | Agentic control-flow y orquestación | Dataflow confiable y pipelines |
| **Key architecture** | Stateful, graph-based runtime; cycles, persistence, human-in-the-loop | Explicit, DG-based pipeline |
| **Strengths** | Workflows multi-step stateful; agentic loops controlables; durable state/checkpointing | RAG medible/auditable/high-perf; estabilidad enterprise; hybrid retrieval; pipelines custom |
| **Ideal use case** | El "brain" de un agente complejo; support bots, research agents multi-step | Tool confiable y deployable (RAG-as-a-service microservice) |
| **2025 v1.0 innovation** | LangGraph runtime | Hayhooks deployment framework |
| **Context management** | Vía middleware en LangChain 1.0 | Vía componentes explícitos del pipeline |

*Table 2.2 – Framework Specialization in 2025 (LangGraph vs. Haystack)*

> [!tip] **¿Por qué no un solo framework?** Se usa el right tool for the job: **Haystack** para la tool layer (sus DGs explícitos, components de hybrid retrieval y evaluation nodes son best-in-class para RAG auditable; y permite desplegar pipelines como microservicios); **LangGraph** para la orchestration layer (su runtime stateful y cíclico es purpose-built para el control-flow persistente que el context engineering requiere). Forzar a un framework a hacer el trabajo del otro genera fricción. El libro argumenta que las **arquitecturas híbridas** serán un patrón de diseño dominante.

### Workflow end-to-end híbrido

1. El usuario hace una pregunta → main agent (powered by LLM).
2. El agente decide que necesita info externa y llama a un tool que busca en la knowledge base.
3. El tool envía la pregunta a la API (Haystack pipeline desplegado), que corre un RAG pipeline completo.
4. El RAG pipeline (microservicio) hace: (a) convierte la pregunta en embedding, (b) encuentra los documentos más relevantes, (c) construye un prompt útil, (d) pide a otro LLM crear una respuesta basada en facts.
5. La API devuelve el resultado en JSON limpio.
6. El tool retorna el resultado al agente.
7. El agente tiene dos cosas: (a) la pregunta original, (b) el contexto recuperado del RAG tool.
8. Usando ambos, el agente escribe la respuesta final.

> [!note] La diferencia clave vs LangChain solo: el RAG tool no es un objeto Python naive leyendo de memoria, sino un **scalable service** capaz de consultar y procesar miles de documentos en varios formatos (text, tabular, audio, image).

## Data storage, retrieval y memory: el rol de los vector stores

### Definición formal y funcionalidad 2025

Un **vector database / vector store** es un sistema de base de datos especializado para almacenar, gestionar e indexar **datos vectoriales de alta dimensión** (IBM, 2025). A diferencia de las DBs relacionales/NoSQL (datos escalares: texto, números, JSON), almacena los datos como vectores matemáticos de longitud fija — los **embeddings**, representaciones numéricas generadas por un embedding model que capturan el significado semántico de datos no estructurados.

> [!note] La función primaria de un vector database **NO es exact match querying**: implementa algoritmos **approximate nearest neighbor (ANN)**, recuperando registros por similitud conceptual/semántica en el espacio de alta dimensión. Es el motor del semantic search moderno. **Gartner (2024)** predijo que para 2026 más del **30% de las empresas** habrán adoptado vector DBs para groundear sus modelos.

### Vector stores como base del RAG empresarial

El language model tiene conocimiento **paramétrico** (aprendido en training); el vector store provee una base **no-paramétrica**, externa y up-to-date — la premisa central de [[RAG]]. Antes de consultarse, el store debe poblarse: la **ingestion pipeline** (= **indexing**) es el componente más crítico y often-failed de la tool layer; un partitioning o chunking mal diseñado daña irreversiblemente la performance.

La ingestion/indexing pipeline (que Haystack está purpose-built para gestionar), en pasos secuenciales:

1. **Partitioning y preprocessing** — transformar raw data (PDFs, HTML, `.txt`) en formato limpio y consistente: extraer texto, remover artifacts, particionar documentos grandes en segmentos lógicos (ej. por section headers).
2. **Chunking** — *la decisión estratégica más crítica del pipeline*. Romper los datos en chunks que respeten el límite de tokens del embedding model (ej. **8,192 tokens** para `text-embedding-ada-002`). Técnicas: **fixed-size chunks** (tamaño fijo ej. 200 words + overlap 10-15%) o **variable-sized chunks** (content-aware, parten en límites semánticos como fin de oración o párrafo).
3. **Embedding** — cada chunk pasa por un embedding model (ej. `text-embedding-ada-002`) que lo convierte en un vector que captura su significado.
4. **Indexing** — cargar los vectores + su texto (el chunk) en el vector database, que construye índices eficientes para búsqueda rápida. (Pasos 1-4 = el **ingest pipeline**.)

Y el segundo pipeline, **retrieval**:

5. **Embedding the query** — la pregunta se transforma a vector **con el MISMO embedding model usado en indexing**, para detectar con precisión vectores semánticamente similares.
6. **Retrieving** — usa la query embebida y fetchea los embeddings más cercanos en el store; devuelve los chunks más relevantes.
7. **Augmenting** — los chunks se pasan como contexto junto a la query original al LLM, que da la respuesta final en lenguaje natural.

> [!warning] **El dilema del chunking — el "semantic pixel size".** RAG recupera *chunks*, no el documento original; el LLM en la generación solo ve esos chunks. Un chunk **muy grande** (ej. un doc de 50 páginas) trae ruido semántico que contamina el context window (análogo a la **context pollution** de los loops ágenticos); uno **muy chico** (una oración) carece del contexto para entender el fact. La chunking strategy (ej. 200 words con 15% overlap) **hardcodea el "semantic pixel size"** de toda la knowledge base, y todo retrieval futuro queda restringido por esa decisión conjunta (chunking + embedding model). Por eso la tool layer debe ser un data pipeline robusto.

### Advanced retrieval: hybrid search

La pure semantic search (dense vectors) tiene una debilidad conocida: **falla en keywords exactos** (ej. un product ID, si el contexto semántico circundante no matchea fuerte). El keyword search tradicional (sparse vectors, **[[BM25]]**/TF-IDF) tiene el problema opuesto: excele en keywords exactos pero tiene **cero comprensión** de la query.

La solución es el **hybrid search**: correr ambas queries en paralelo (una contra el dense vector index, otra contra el sparse/BM25), y fusionar los dos result sets en un ranking superior. El algoritmo más común y efectivo es **[[Reciprocal Rank Fusion (RRF)|Reciprocal Rank Fusion (RRF)]]**, que re-rankea según la posición en las listas originales, evitando normalizar scores incompatibles.

> [!tip] El hybrid search es un **baseline requirement de RAG production-grade**. Haystack 2.0 encapsula todo el data-flow (ingestion, chunking, embedding, indexing, retrieval) en **dos tools deployables** — un **indexing pipeline** y un **retrieval pipeline** — cada uno microservicio independiente vía [[Hayhooks]]/REST. La orchestration layer no necesita saber cómo funciona el retrieval: solo llama el RAG tool y recibe un JSON limpio con el contexto grounded.

### Evolución: de RAG index a memoria ágentica

El rol del vector store evolucionó de **fuente de datos estática read-only** (2023) a **capa de memoria dinámica read-write** (2025). En 2023 era pasivo (índice no-paramétrico de docs estáticos para groundear outputs; write-once read-many). Pero los sistemas ágenticos de 2025 corren en loops stateful y generan datos en cada ciclo, mientras los LLMs son **stateless** con context windows finitos. La solución: arquitecturar la memoria en capas — **short-term** (context window inmediato), **working memory** (scratchpad del agente) y **long-term** (store externo persistente = el vector database).

Esta vector-based long-term memory **reutiliza el mecanismo RAG**: en vez de recuperar facts externos, el agente recupera sus **propias experiencias** (conversation history, knowledge, task info como vectores buscables), permitiéndole recordar interacciones, aprender y comportarse consistentemente.

### El vector store como enabler de context engineering

El vector store es la **implementación física** de las estrategias **write** y **select**:

- **Write strategy** — almacenar info externamente para retrieval posterior. Es lo que Anthropic (2025) llama *structured note-taking* / agentic memory: tras una interacción, el agente escribe info crítica (preferencias nuevas, summaries de tareas en curso, conclusiones intermedias) como embedding en el vector database, persistiéndola fuera del context window.
- **Select strategy** — traer dinámicamente solo la info más relevante al context window (memory retrieval): al empezar su próximo reasoning step, el agente consulta el vector store (su long-term memory) con su objetivo actual, y las memorias más relevantes se inyectan al prompt. Reduce el **context rot**.

> [!warning] **Memory curation (problema de tercer orden).** El patrón read-write crea un problema nuevo: si el agente escribe constantemente, el store se llena de info **redundante, conflictiva y trivial** → context pollution, y el select empieza a recuperar contexto de baja calidad. Es el mismo problema que la disciplina buscaba resolver.

La solución AWS (Sehwag et al. 2025, ej. Amazon **AgentCore**) es un **consolidation process** (loop recursivo) que trata la memoria como un asset curado:

1. Cuando el write strategy genera una memoria nueva, **NO** se commitea directo; el sistema primero hace **retrieval** de las memorias existentes más semánticamente similares.
2. La memoria nueva pendiente + las viejas recuperadas se **bundlean** y se mandan a un LLM con un **consolidation prompt**.
3. El LLM actúa como **database manager** y decide la acción: `ADD` (info novel), `UPDATE` (refina/fusiona con memoria existente) o `NO-OP` (redundante).
4. Solo entonces se ejecuta la acción (ej. actualizar el vector store).

> [!note] Este loop stateful y self-curating es un hallmark del agente de 2025. Acá una orchestration layer desacoplada se vuelve especialmente útil: el state management, middleware y la definición graph-based de [[LangGraph]] pueden implementar este patrón.

## La economía del deployment: el costo de inference

La especialización en LLMs/SLMs/RLMs no es solo arquitectónica sino **profundamente económica**. Hay que diferenciar formalmente los costos de **training** vs **inference** y entender el **TCO** (total cost of ownership).

> [!warning] **Misconception común: que el costo primario es el training. Es incorrecto.** Para developers y empresas en 2025, el costo dominante, recurrente y estratégico es la **inference**.
> - **Training cost** — CAPEX masivo y único para crear el foundational model (analogía: *"everything that goes into building a car"*); requiere clusters de GPUs costando miles de millones. Dominio exclusivo de las AI foundries (OpenAI, Google, Anthropic).
> - **Inference cost** — OPEX recurrente para usar el modelo (*"inference is the cost to operate the cars, like gasoline"*). Para la mayoría de las organizaciones (AI consumers, no producers), es el costo que dicta la viabilidad económica.

El panorama económico de inference de 2025 tiene una paradoja: mientras el costo de training sube, el unit cost de inference está en **caída libre** — **"The Great AI Cost Compression of 2025"**, una API price war entre incumbentes (OpenAI, Google), una ofensiva open-source (Meta, Mistral) y startups inference-as-a-service.

### TCO y el latency-throughput trade-off

Lo que importa al arquitecto es el **TCO**, no solo el precio de comprar/llamar al modelo. Para API-based es directo (cost de input tokens + output tokens). Para self-hosted es complejo (hardware benchmarking, energía, networking, ingeniería continua).

> [!note] **El latency-throughput trade-off** (Nvidia, 2025) es un factor mayor del TCO. Aumentar el **throughput** (volumen de requests, ej. vía batching) **siempre aumenta la latency** (la velocidad que el usuario experimenta). El trabajo del arquitecto: **definir una latency máxima aceptable y maximizar throughput dentro de esa restricción** — lo que determina el número óptimo de instancias y, con ello, el costo total.

El rol de los **SLMs** en el TCO: un SLM de **7B params es 10-30× más barato de servir** que uno de 70-175B (Nvidia Research, Belcak et al. 2025), por performance más rápida, menor energía y menos compute.

> [!tip] **TCO crossover / breakeven point** (Pan et al. 2025). Los modelos API tienen bajo costo inicial pero pricing usage-based (crece y se vuelve impredecible a escala); los self-hosted son más caros inicialmente pero predecibles. A cierto nivel de uso, el costo acumulado de las API calls **supera** al de poseer infraestructura propia — ese punto de cruce es central para decidir cuándo conviene self-hostear.

### La decisión capstone: private vs hosted

La decisión final sintetiza lo técnico y financiero en lo estratégico: un trade-off entre **soberanía y conveniencia**, impulsado por **data security, model customization y performance**.

#### Tabla 2.3 — Decision matrix: private vs hosted deployment

| Criterio | Private deployment | Hosted deployment |
|---|---|---|
| **Data security / compliance** | Control total, datos no salen del entorno; requerido para HIPAA, GDPR, SOC 2 | Riesgo: datos a un tercero (pueden usarse para training); no apto para datos altamente regulados |
| **Model customization** | Control total; fine-tuning parameter-level vía PEFT/LoRA; modelos especializados | Limitado a prompt engineering (costoso) o fine-tuning APIs del vendor |
| **Performance (latency)** | Baja y predecible (decenas de ms); requerido para edge AI y real-time UI | Alta y variable (multi-segundo, según red y carga del provider) |
| **TCO (volumen bajo/irregular)** | Muy caro, no amortizado; dominan los idle hardware costs ("wall-clock") | Ideal, pay-as-you-go, cero costo idle |
| **TCO (volumen alto/consistente)** | Cost-effective, amortizado sobre muchas llamadas, más barato que API | Muy caro, escala linealmente con el volumen de tokens |
| **MLOps burden** | Muy alta; equipo especializado para hardware/utilización/optimización | Nula; el provider gestiona toda la infra y MLOps |
| **Best-fit use case** | Tareas seguras, low-latency, high-volume o especializadas (on-device, internal tools, core features) | Razonamiento general, prototyping, low-volume, non-core features |

*Table 2.3 – Decision matrix: private vs. hosted deployment*

> [!tip] Esta matriz enmarca el desafío arquitectónico core como un trade-off entre los **cuatro pilares del deployment**: **capability, cost, latency y privacy**. Conclusión: *"In 2026 and beyond, winning architectures aren't defined by model size or novelty but by economic clarity and the disciplined alignment of technical decisions with real, recurring cost structures."*

## Citas

> "For developers, this means the end of the one-size-fits-all era; architects must now choose models based on the specific trade-offs between operational cost, latency, and cognitive depth."
> "the delicate art and science of filling the context window with just the right information for the next step"
> "It is the art and science of curating what will go into the limited context window from a constantly evolving universe of possible information"
> "This architecture is not LangGraph or Haystack but LangGraph and Haystack."
> "a set of hooks that allow you to customize behavior in the agent loop, enabling fine grained control at every step an agent takes"
> "inference is the cost to operate the cars, like gasoline"
> "Research shows that a 7-billion-parameter SLM is 10–30× cheaper to serve than a much larger model with 70–175 billion parameters"
> "In 2026 and beyond, winning architectures aren't defined by model size or novelty but by economic clarity and the disciplined alignment of technical decisions with real, recurring cost structures."

## Para aplicar

- **Setup del capítulo** — `uv` para paquetes, Python 3.11+; clonar el repo y trabajar en `ch2`:
  ```bash
  git clone https://github.com/PacktPublishing/Building-Natural-Language-and-LLM-Pipelines.git
  cd building-natural-language-and-LLM-pipelines/
  cd ch2
  uv sync
  source .venv/bin/activate
  ```
  Kernel Jupyter `rag-with-haystack-ch2` (path `.venv/bin/python`); crear `.env` (OpenAI API u Ollama local con `ollama pull <model>`); recomendado Miniconda + VS Code.
- **Elegir arquitectura por tarea** — SLMs para high-volume/cost-sensitive/edge; RLMs para razonamiento multi-paso deliberado; Haystack (tool layer) + LangGraph (orchestration layer).
- **Aplicar las 4 estrategias de context engineering** — Write/Select/Compress/Isolate para gestionar el context window y evitar context rot.
- **RAG production-grade** — implementar **hybrid search con RRF** como baseline; elegir chunking strategy y embedding model con cuidado (decisiones permanentes que hardcodean el semantic pixel size; respetar el límite de tokens del embedder).
- **Memoria ágentica read-write** — implementar un **consolidation process** (ADD/UPDATE/NO-OP) para evitar el memory curation problem; usar LangGraph (state, middleware) para implementarlo.
- **Deployment** — enfocarse en **TCO** (no solo el precio del modelo); definir la latency máxima aceptable y maximizar throughput dentro de esa restricción; calcular el breakeven point self-hosted vs API.
- **Notebooks de práctica** — `01_prompt-ollama-model.ipynb`, `02_create-simple-agent.ipynb`, `03_document-qa-langchain.ipynb`; avanzados (LangGraph): `04_understanding-state-graph.ipynb`, `05_graph-based-agent-with-tools.ipynb`, `06_multi-agent-systems-middleware.ipynb`.

## Conexiones

- [[_Building Natural Language and LLM Pipelines|Building NL & LLM Pipelines]] — el MOC del libro.
- [[01 - Introduction to Natural Language Processing Pipelines]] — capítulo anterior (sienta el tool vs orchestration layer y la agentic reliability crisis que este capítulo profundiza en lo técnico).
- [[03 - Introduction to Haystack by deepset]] — capítulo siguiente (deep dive en la tool layer / Haystack 2.0 introducida aquí).
- [[Context Engineering]] — disciplina central, automatizada como ACE en [[09 - Future Trends and Beyond]] y mapeada a V3 en el [[10 - Epilogue - The Architecture of Agentic AI|Epílogo]].
- [[Haystack 2.0]] · [[LangGraph]] · [[Hayhooks]] — los frameworks de la arquitectura híbrida.
- [[RAG]] · [[Hybrid Search]] · [[BM25]] · [[Reciprocal Rank Fusion (RRF)]] · [[Embeddings]] · [[Transformer]] · [[BERT]] — conceptos retomados en [[04 - Bringing Components Together - Haystack Pipelines for Different Use Cases]] y [[06 - Building Reproducible and Production-Ready RAG Systems]].
- [[DeepSeek-R1]] · [[GRPO]] — modelos open-weight, reaparecen en [[09 - Future Trends and Beyond]] y el Epílogo (stress-test).
- [[TCO (Total Cost of Ownership)]] · [[Sovereign agent]] — economía de inference, base del sovereign stack del Epílogo.
- [[Model Context Protocol (MCP)]] — despliegue de pipelines como tools, detallado en [[07 - Deploying Haystack-Based Applications]].
- **SLM** · **RLM** · **context rot** · **memory curation** · **TCO crossover point** — conceptos clave del capítulo; candidatos a nota propia.
