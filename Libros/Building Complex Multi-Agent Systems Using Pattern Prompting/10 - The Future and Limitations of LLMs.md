---
title: The Future and Limitations of LLMs
libro: Building Complex Multi-Agent Systems Using Pattern Prompting
autor: Packt
capitulo: 10
created: 2026-06-22
tags:
  - libros/building-complex-multi-agent-systems-using-pattern-prompting
  - type/case-study
  - status/permanent
aliases:
  - The Future and Limitations of LLMs
  - Cap 10 - The Future and Limitations of LLMs
updated: 2026-06-22
---

# The Future and Limitations of LLMs

> [!info] Capítulo 10 · *Building Complex Multi-Agent Systems Using Pattern Prompting* — Packt (ISBN 9781806114290) · **ÚLTIMO capítulo del libro**
> El capítulo de cierre es **conceptual, sin código**: su sustancia son los argumentos, las citas verbatim y la **Table 10.1**. La tesis: distinguir **lo que los [[LLM|LLMs]] realmente hacen** (predecir el próximo token) de los claims de hype (AGI). Se puede conocer el límite de la AI porque coincide con el límite de la **[[Computabilidad|computabilidad]]** (Gödel, Turing, Russell, Penrose); aceptar AGI sería paradójico porque socava la ciencia sobre la que se construye. Recorre los 4 argumentos clásicos —**[[Stochastic Parrot|stochastic parrot]]** (Bender et al. 2021), **[[Chomsky]]** (Universal Grammar), **[[Chinese Room]]** (Searle) y **[[Gödel incompleteness|Gödel]]/Penrose**—, el **reasoning gap** (5 cosas que el LLM no puede hacer fiablemente, incluido el **[[Symbol Grounding Problem|symbol grounding]]**), por qué el hype tiene **costos de ingeniería** reales, y para qué los LLMs son **genuinamente buenos** (tareas con output verificable; **[[RAG]]** como el deployment pattern más maduro porque constriñe al LLM a un dominio verificable — la microarquitectura del [[01 - Introduction Patterns, Abstractions, and the GenAI Landscape|cap. 1]], [[04 - Building Your First RAG App|cap. 4]] y [[08 - Pattern-Guided Coding|cap. 8]]). Cierra con **6 principios de ingeniería** y la postura del buen ingeniero. Navegá: [[_Building Complex Multi-Agent Systems Using Pattern Prompting|Building Complex Multi-Agent Systems]] · anterior [[09 - Implementing the ReAct Pattern Over RabbitMQ]] · (último capítulo).

## Resumen

El capítulo final del libro es deliberadamente **conceptual** y funciona como un **antídoto contra el hype**: su objetivo es darle al lector un **respeto calibrado** ("calibrated respect") por los [[LLM|LLMs]] y la inmunidad necesaria para no confundir la **fluidez** con el **entendimiento**. La tesis central es que se puede **conocer el límite de la AI** con cierta precisión, porque ese límite coincide con el límite de la **[[Computabilidad|computabilidad]]** misma —el territorio mapeado por **Gödel, Turing, Russell y Penrose**—. Aceptar la **AGI** (Artificial General Intelligence) como inevitable sería paradójico: socavaría la lógica, la matemática y la física sobre las que la propia ciencia (y la propia AI) se construye. El capítulo invoca a **Minsky** ("the brain is just a meat computer") y advierte que los thought leaders contemporáneos (**Hinton, Musk, Bengio, Ng**) emiten opiniones que **no cargan el peso de justificación** de sus predecesores; el principio rector es **"a theory put forward without justification can be rejected without justification"**.

El núcleo descriptivo es preciso: un LLM es una **red neuronal entrenada para predecir el próximo token**, y **todo lo demás** —traducción, summarización, question-answering, código, "razonamiento"— es **emergente** de esa única objective function. El LLM **no recupera hechos**, **no consulta creencias**, **no corre simulaciones** y **no aplica lógica**: acierta cuando la respuesta correcta estaba **bien representada en el training data**. Sobre esta base, el capítulo despliega **cuatro argumentos** filosófico-técnicos que delimitan lo que los LLMs no son —el **[[Stochastic Parrot|stochastic parrot]]** (Bender et al. 2021), la crítica de **[[Chomsky]]** desde la adquisición humana del lenguaje, el **[[Chinese Room|Chinese Room]]** de Searle, y los **límites matemáticos** de Gödel reformulados por **Penrose**— y luego el **reasoning gap**: las **5 cosas** que los LLMs no pueden hacer de forma fiable (razonamiento matemático sistemático, consistencia lógica en contextos largos, confianza calibrada, insights novedosos y **[[Symbol Grounding Problem|symbol grounding]]**).

La segunda mitad es práctica para el ingeniero: **qué dice el hype y cuál es la realidad técnica** (la **Table 10.1** de 6 claims), **por qué el hype es dañino** no como cuestión retórica sino por **5 costos de ingeniería** concretos, **para qué los LLMs son genuinamente buenos** (tareas donde el output se **verifica independientemente** y el costo de error ocasional es aceptable; **[[RAG]]** como el deployment pattern más maduro porque trata al LLM como **interfaz de lenguaje, no oráculo**), y cómo **pensar el futuro con ojos claros** (honestidad intelectual en ambas direcciones: los críticos pueden equivocarse, pero los promotores de AGI deben rendir cuentas — "extraordinary claims require extraordinary evidence"). Termina con **6 principios de ingeniería** accionables y un **Summary** que cierra el libro entero: los LLMs son poderosos y transformadores para **una clase específica de tareas**, pero **pattern-matchers estocásticos** que no entienden, no razonan sobre lo novel y alucinan con confianza — y esas son **propiedades estructurales, no bugs temporales**.

## What LLMs actually do

El capítulo arranca anclando la discusión en una **descripción precisa** de lo que es un LLM, despojada de antropomorfismo. Un LLM es una **red neuronal entrenada con una única objective function: predecir el próximo token** dado un contexto. No hay un segundo módulo "de razonamiento", ni una base de datos de hechos, ni un motor de inferencia lógica: hay una distribución de probabilidad sobre el próximo token, aprendida de un corpus masivo.

> [!note] **Todas las capacidades del LLM son emergentes de la predicción del próximo token.** Traducción, summarización, question-answering, generación de código y la apariencia de "razonamiento" no son funciones programadas: emergen de optimizar esa única objective function sobre suficiente data y suficientes parámetros.

La consecuencia es directa y desmitificadora: el LLM **no recupera hechos** de una memoria, **no consulta creencias**, **no corre simulaciones** mentales y **no aplica reglas de lógica**. Lo que hace es producir el texto **más probable** dado el contexto.

> [!warning] Un LLM **acierta cuando la respuesta correcta estaba bien representada en el training data** —y la fluidez del output es la misma esté la respuesta bien o mal representada—. Esto explica por qué un LLM puede ser brillante en lo común y confiadamente erróneo en lo raro: la **plausibilidad lingüística** del output no está acoplada a su **veracidad**.

Sobre esta base, el capítulo introduce su tesis epistemológica: **el límite de la AI es cognoscible** porque coincide con el límite de la **[[Computabilidad|computabilidad]]**. El trabajo de **Gödel, Turing, Russell y Penrose** mapeó qué puede y qué no puede hacer cualquier sistema formal/algorítmico; un LLM es un sistema computacional, así que hereda esos límites. Aceptar la **AGI** como un hecho inminente sería **paradójico**: la ciencia que produciría la AGI descansa en la lógica, la matemática y la física, y varios de los argumentos del capítulo sugieren que **la cognición humana excede lo algorítmico** —de modo que afirmar que una máquina algorítmica la igualará socava las propias herramientas con que se hace la afirmación—.

> [!note] **Minsky y el "meat computer".** El capítulo cita la postura de **Marvin Minsky** de que el cerebro "is just a meat computer" —el polo materialista/computacionalista de la discusión—. La presenta no para refutarla de plano, sino para contrastarla con los argumentos que sugieren que la cognición humana **no** es reducible a cómputo.

> [!tip] **El peso de la justificación.** Los thought leaders contemporáneos —**Geoffrey Hinton, Elon Musk, Yoshua Bengio, Andrew Ng**— hacen predicciones audaces sobre la AGI, pero el capítulo advierte que sus opiniones **no cargan el peso de justificación** que tenían las de los predecesores (Gödel, Turing). El principio operativo: **"a theory put forward without justification can be rejected without justification."**

### The stochastic parrot

El primer argumento es el del **[[Stochastic Parrot|stochastic parrot]]**, de **Emily Bender, Timnit Gebru, Angelina McMillan-Major y Shmargaret Shmitchell (2021)** en *On the Dangers of Stochastic Parrots*. La idea: el LLM produce **fluidez sin entendimiento**. Cose secuencias de formas lingüísticas que observó en su training data según información probabilística de cómo se combinan, **sin referencia alguna al significado**.

> [!note] La consecuencia más peligrosa del stochastic parrot: produce texto **autoritativo y fluido aun cuando está factualmente mal**, y **no tiene mecanismo para saber que no sabe**. No hay un "estado de incertidumbre" interno acoplado a la verdad —solo a la probabilidad lingüística—.

### Chomsky: machine learning es antitético a la adquisición del lenguaje

El segundo argumento viene de **Noam Chomsky**, con **Ian Roberts y Jeffrey Watumull**, en un artículo del **New York Times (2023)**. La tesis: el enfoque de **machine learning** para la adquisición del lenguaje es **fundamentalmente distinto y antitético** al de la adquisición humana.

- **Universal Grammar.** Los chicos aprenden lenguaje a partir de **input escaso** (poverty of the stimulus): pocos ejemplos, muchos de ellos imperfectos. Lo logran porque traen una **[[Universal Grammar|Universal Grammar]] innata** —una estructura biológica que restringe las gramáticas posibles—. Los LLMs, en cambio, necesitan corpus masivos.
- **No distinguen gramatical de agramatical por regla.** Un LLM **no puede** decir que una oración es agramatical por una **regla**; solo refleja la **distribución** del training data. Una oración como **"Store the to going I am"** es rechazable por un humano por estructura, no por frecuencia.
- **Verdadera inteligencia = razonar sobre lo posible y lo imposible.** Para Chomsky, la inteligencia genuina razona sobre **hipotéticos** y **contrafácticos** —lo que podría ser y lo que no podría ser—, no solo sobre lo que es estadísticamente frecuente.

> [!note] Chomsky concluye que ChatGPT y similares son **"constitutionally unable"** de decirnos algo sobre **qué es posible en el lenguaje y el pensamiento humano**: describen la distribución, no la facultad.

### The Chinese Room

El tercer argumento es el experimento mental del **[[Chinese Room|Chinese Room]]** de **John Searle**. Una persona encerrada en un cuarto recibe símbolos en chino por una ranura, consulta un **rulebook** (un libro de reglas) que le dice qué símbolos devolver, y produce respuestas en chino correctas **sin entender una palabra de chino**.

> [!note] El paralelo con el LLM es exacto: la persona **manipula símbolos (tokens) según reglas (weights)** sin acceder a la **semántica**. El argumento de Searle: **la sintaxis no es suficiente para la semántica** —"syntax is not sufficient for semantics"—. La **apariencia** de entender (output indistinguible del de un hablante real) **no es** entender.

### Penrose, Gödel y los límites matemáticos

El cuarto y más profundo argumento viene de **Roger Penrose** en *The Emperor's New Mind* (**1989**) y *Shadows of the Mind* (**1994**), y se apoya en el **teorema de incompletitud de [[Gödel incompleteness|Gödel]] (1931)**.

- **Gödel (1931).** Todo **sistema formal consistente** lo bastante potente como para expresar aritmética contiene **verdades que no puede probar** dentro del sistema. Hay enunciados verdaderos pero formalmente indecidibles ("gödelianos").
- **El paso de Penrose.** Los **matemáticos humanos** captan por **insight** que ciertos enunciados gödelianos son **verdaderos**, aunque caigan **fuera del alcance demostrativo** de cualquier sistema formal dado. Si la cognición matemática humana fuera algorítmica, estaría confinada a un sistema formal y **no podría** captar esas verdades. Por lo tanto —concluye Penrose— **la cognición matemática humana no es algorítmica**.
- **Orch-OR (más especulativo).** Penrose va más lejos con la hipótesis **Orch-OR** (orchestrated objective reduction): procesos **cuánticos** en los **microtúbulos** de las neuronas serían la fuente de esa cognición no-computable. Esta parte es **más especulativa y contestada**.

> [!warning] El capítulo separa los dos niveles: la hipótesis **Orch-OR** es discutida y no necesaria para el argumento. **El punto gödeliano por sí solo** —que la cognición matemática humana parece exceder lo capturable por cualquier conjunto de reglas— **ya es significativo** y suficiente para el caso del capítulo.

## The reasoning gap

Más allá de los argumentos filosóficos, el capítulo enumera **5 cosas concretas** que los LLMs **no pueden hacer de forma fiable**. No son fallas de implementación de un modelo particular: son **propiedades estructurales** del enfoque.

> [!note] **1 · Razonamiento matemático sistemático.** Los LLMs **colapsan al reformular** las variables o la estructura de un problema. Pueden resolver un problema que vieron, pero un cambio cosmético en el planteo (renombrar variables, alterar la estructura superficial) suele derrumbar el desempeño —señal de que están **reconociendo el patrón del problema**, no aplicando el procedimiento—.

> [!note] **2 · Consistencia lógica en contextos largos.** En contextos extensos los LLMs **se contradicen**. Es una consecuencia del **[[Attention Mechanism|attention mechanism]]**: la atención sobre tokens distantes se diluye, así que afirmaciones lejanas en el contexto dejan de restringir lo que el modelo genera.

> [!warning] **3 · Confianza calibrada.** El LLM mantiene un **tono confiado siempre**, esté en lo correcto o no. La **[[Hallucination|hallucination]] NO es un bug**: es la **consecuencia directa del training objective**, que premia **fluidez y plausibilidad**, no **verdad**. Un texto inventado puede ser tan plausible —y por tanto tan probable— como uno correcto. No hay señal interna de "esto lo estoy inventando".

> [!note] **4 · Insights novedosos.** Los LLMs son buenos en **interpolación** (combinar y rellenar dentro del espacio de su training data) pero **limitados en extrapolación** (salir genuinamente de ese espacio). Lo que se percibe como "creatividad" es **recombinación**, no **invención**.

> [!note] **5 · Symbol grounding problem.** El **[[Symbol Grounding Problem|symbol grounding problem]]** (Harnad, **1990**): los símbolos de un LLM solo se definen **circularmente** —cada palabra se define con otras palabras—. Entender **"red"** ("rojo") requiere haber **percibido** la rojez; los LLMs solo tienen **texto**, nunca percibieron nada. Es un problema **estructural, no de escala** (más data no lo resuelve). Por eso los LLMs razonan **mejor** sobre lo que está muy descripto en texto y **peor** sobre la intuición física *grounded* (lo que un humano sabe por haber tenido un cuerpo en el mundo).

> [!tip] El reasoning gap es la cara operativa de los cuatro argumentos: stochastic parrot → confianza no calibrada; Chomsky → no razonar sobre lo posible/imposible; Chinese Room y symbol grounding → sintaxis sin semántica/grounding; Gödel/Penrose → matemática sistemática fuera de alcance.

## What the hype gets wrong

El capítulo confronta los **claims del hype** con su **realidad técnica**. Cada afirmación marketinera tiene una contraparte precisa que el ingeniero necesita tener presente. La **Table 10.1** las reúne.

### Tabla 10.1 — Claim vs. technical reality

| Claim (hype) | Technical reality |
|---|---|
| **"The model understands the document"** | The model has computed a statistical representation of the text and can produce plausible continuations. There is no comprehension —no internal model of meaning that the text refers to—; it manipulates linguistic form according to learned distributions. |
| **"The model reasons about the problem"** | The model produces text that resembles reasoning because reasoning-shaped text was in its training data. It is **pattern-matching the form of an argument**, not executing a logical procedure; reformulating the problem can break it. |
| **"The model learned from the data"** | The model adjusted billions of weights to better predict the next token over the corpus. It did not "learn" in the human sense of forming beliefs or understanding; it captured statistical regularities. |
| **"The model knows X"** | The model can produce text asserting X when X was well represented in training. It has **no beliefs and no truth-tracking mechanism**: it will assert X or not-X with equal fluency depending on the prompt and the distribution, with no way to know which is true. |
| **"The model is creative"** | The model recombines and interpolates within the space of its training data. This can be genuinely useful, but it is **recombination, not invention**: it does not extrapolate to genuinely novel insight outside that space. |
| **"AGI is two years away"** | An extraordinary claim with **no justification** that survives the arguments above. Scaling next-token prediction does not address symbol grounding, calibrated truth, or the Gödelian limits; these are **structural**, not engineering details that more compute removes. |

> [!warning] La tabla deja explícito el **patrón del hype**: cada claim toma un verbo humano —understand, reason, learn, know, create— y lo aplica a una operación que es, en realidad, **manipulación estadística de forma lingüística**. El antídoto no es el cinismo, sino **nombrar la operación real** en cada caso.

## Why hype is harmful

El capítulo argumenta que el hype no es solo un problema retórico: tiene **costos de ingeniería** concretos y medibles. Lista **cinco**.

> [!warning] **Los 5 costos de ingeniería del hype**
> - **Misallocated reliability budget** — si creés que el modelo "entiende", subinvertís en verificación, validación y fallbacks justo donde más hacen falta. El presupuesto de fiabilidad se asigna mal.
> - **Wrong evaluation criteria** — evaluás el sistema por la fluidez/plausibilidad de su output (que siempre es alta) en vez de por su **veracidad** o su comportamiento en los casos difíciles; los benchmarks miden lo que el modelo hace bien por construcción.
> - **Vendor lock-in under false pretenses** — comprás capacidades que el sistema no tiene realmente (razonamiento, conocimiento), atándote a un provider sobre premisas falsas (eco del **[[Vendor Lock-in|vendor lock-in]]** del [[07 - Tips and Best Practices|cap. 7]]).
> - **Regulatory and liability exposure** — desplegás en dominios de alto riesgo (legal, médico, financiero) confiando en "entendimiento" inexistente, exponiéndote a responsabilidad regulatoria y legal cuando el modelo alucina con confianza.
> - **Erosion of professional judgment** — el costo **más insidioso y cultural**: los profesionales delegan su juicio en un sistema que parece autoritativo, y con el tiempo **atrofian la propia expertise** que les permitiría detectar cuándo el modelo se equivoca. Es el que más cuesta revertir.

## What LLMs are genuinely good at

Tras delimitar lo que los LLMs no son, el capítulo es igual de claro sobre **dónde sí aportan valor real**. El criterio es preciso: tareas donde el **output se verifica independientemente** y el **costo de un error ocasional es aceptable**.

> [!note] **El sweet spot de los LLMs:** drafting (borradores), traducción, summarización, generación de **código a partir de requisitos bien especificados**, extracción de **datos estructurados** y boilerplate. En todas, el humano (o un sistema) **verifica** el output, y un error de vez en cuando no es catastrófico. Un beneficio transversal: eliminan el **"blank-page problem"** —el costo de arrancar de cero—.

El deployment pattern más maduro es **[[RAG]]** (retrieval-augmented generation), y el capítulo explica **por qué** en términos de este libro:

> [!tip] **RAG como microarquitectura que constriñe al LLM a un dominio grounded y verificable.** RAG trata al LLM como una **interfaz de lenguaje, no como un oráculo**: en vez de pedirle al modelo que "sepa" la respuesta (donde alucina), le da el **contexto recuperado** y le pide que lo **reformule/sintetice** —una tarea de las que sí hace bien, con la fuente disponible para verificar—. Es la misma **[[RAG]]** que el [[01 - Introduction Patterns, Abstractions, and the GenAI Landscape|cap. 1]] presentó como microarquitectura, el [[04 - Building Your First RAG App|cap. 4]] construyó production-grade sobre [[RabbitMQ]], y el [[08 - Pattern-Guided Coding|cap. 8]] extendió a dual-LLM. El capítulo la nombra **explícitamente** como el patrón más maduro precisamente porque **constriñe al LLM a un dominio verificable**.

> [!note] **La postura productiva** que el capítulo propone: tratar al LLM como un **asistente capaz pero poco fiable** cuyo output **siempre se verifica**, y **diseñar la arquitectura sobre ese supuesto desde el día 1** —no como un parche posterior—. Es la continuidad directa del **LLM como endpoint poco fiable** del [[02 - Embeddings The Language of AI|cap. 2]] y del **poison message** del [[09 - Implementing the ReAct Pattern Over RabbitMQ|cap. 9]].

## Thinking about the future with clear eyes

El capítulo insiste en la **honestidad intelectual en ambas direcciones** —ni entusiasmo acrítico ni rechazo reflexivo—.

- **Los críticos pueden estar equivocados.** Chomsky, Penrose, Searle y Bender plantean límites que **podrían** resultar problemas de **ingeniería** (solubles con mejores arquitecturas) más que **fundamentales**. El capítulo no los presenta como verdad revelada.
- **Pero los promotores de AGI también deben rendir cuentas.** El estándar es simétrico: **"extraordinary claims require extraordinary evidence."** Quien afirma que la AGI está a la vuelta de la esquina carga el peso de la prueba, y ese peso —dados los argumentos del capítulo— no está saldado.

> [!tip] **La posición responsable** no es la del escéptico ni la del creyente, sino la del **buen ingeniero**: tomar en serio tanto las capacidades reales como los límites estructurales, diseñar para la incertidumbre, y mantener el juicio propio afilado. La misma cabeza fría con que el [[07 - Tips and Best Practices|cap. 7]] pedía evaluar la "irrational/frothy exuberance".

## Practical guidance

El capítulo aterriza todo en **6 principios de ingeniería** para construir con LLMs sabiendo lo que son.

> [!tip] **Los 6 principios**
> - **Always verify factual claims** — usá **[[RAG]] con fuentes citadas** como baseline, y **human review** en aplicaciones de alto riesgo. Nunca confíes en que el modelo "sabe".
> - **Design for hallucination, not against it** — asumí que el modelo alucinará y construí la arquitectura alrededor de ese hecho (verificación, fallbacks, validación), en vez de esperar eliminarla con prompting.
> - **Test at the edge of the distribution** — testeá en los **casos raros y reformulados**, no en los comunes que el modelo hace bien por construcción; ahí es donde aparece el reasoning gap.
> - **Separate generation from judgment** — **no le pidas al LLM que evalúe su propio output**. Generación y juicio deben ser pasos (o sistemas) distintos; un modelo que se autoevalúa hereda sus propios sesgos.
> - **Communicate uncertainty to users** — exponé al usuario que el output es probabilístico y verificable, no autoritativo; no presentes la respuesta del modelo como un hecho establecido.
> - **Maintain human expertise** — preservá la expertise humana que permite detectar los errores del modelo; es la defensa contra la **erosion of professional judgment** —el costo más insidioso del hype—.

## Summary

> [!note] **Cierre del libro.** Los LLMs son **poderosos y transformadores** para una **clase específica de tareas** —aquellas con output verificable y costo de error aceptable—, pero son **pattern-matchers estocásticos**: **no entienden**, **no razonan sobre lo novel**, **alucinan con confianza** y **no tienen calibración** entre fluidez y verdad. Y estas son **propiedades estructurales, no bugs temporales** que más compute o más data eliminen.

El capítulo recuerda que los críticos rigurosos vienen de tres disciplinas que **definen las boundary conditions** del campo: la **lingüística** ([[Chomsky]]), la **filosofía de la mente** (Searle, **Chalmers**) y la **matemática/física** (Penrose). El lector debe terminar el libro con **"calibrated respect"** por los LLMs e **inmunidad al hype** —la capacidad de usar la herramienta por lo que es, sin confundir la apariencia del pensamiento con el pensamiento—.

## Citas

> "Large language models are systems for haphazardly stitching together sequences of linguistic forms it has observed in its vast training data, according to probabilistic information about how they combine, but without any reference to meaning: a stochastic parrot."
> — Emily Bender, Timnit Gebru, Angelina McMillan-Major, and Shmargaret Shmitchell, *On the Dangers of Stochastic Parrots* (2021)

> "The machine-learning approach to language acquisition is fundamentally different from and antithetical to the approach of human language acquisition. ChatGPT and its brethren are constitutionally unable to tell us anything about what is possible in human language and in human thought."
> — Noam Chomsky, Ian Roberts & Jeffrey Watumull, *The New York Times* (2023)

> "Gödel's theorem is not merely a technical curiosity. It tells us something deep about the nature of understanding — that mathematical insight goes beyond what can be captured in any set of rules, no matter how elaborate."
> — Roger Penrose, *Shadows of the Mind* (1994)

> "The question is not whether machines can think, but whether the thinking they do is the kind of thinking we imagine it to be. The evidence so far suggests we should be much more careful about that distinction than most people are."
> — Reflecting a consensus view among rigorous critics of AI maximalism

## Para aplicar

- **Always verify factual claims** — verificá toda afirmación factual del modelo; **[[RAG]] con fuentes citadas** como baseline y **human review** en alto riesgo.
- **Design for hallucination, not against it** — asumí la [[Hallucination|hallucination]] como dada y diseñá la arquitectura para contenerla (verificación, fallbacks, validación), no para eliminarla con prompting.
- **Test at the edge of the distribution** — testeá en casos raros y **reformulados**, donde el reasoning gap se hace visible, no en los comunes que el modelo acierta por construcción.
- **Separate generation from judgment** — nunca le pidas al LLM que evalúe su propio output; generación y juicio en pasos/sistemas distintos.
- **Communicate uncertainty to users** — presentá el output como probabilístico y verificable, no como un hecho autoritativo.
- **Maintain human expertise** — preservá la expertise humana que detecta los errores del modelo; es la defensa contra la erosion of professional judgment.
- **La postura productiva** — tratá al LLM como un **asistente capaz pero poco fiable cuyo output siempre se verifica**, y diseñá la arquitectura sobre ese supuesto **desde el día 1**; **[[RAG]]** es el patrón más maduro porque constriñe al LLM a un dominio grounded y verificable (interfaz de lenguaje, no oráculo).

## Conexiones

- [[_Building Complex Multi-Agent Systems Using Pattern Prompting|Building Complex Multi-Agent Systems]] — el MOC del libro.
- [[09 - Implementing the ReAct Pattern Over RabbitMQ]] — **capítulo anterior** (y último con código): tras construir un agente ReAct production-shaped, este capítulo de cierre da el **marco conceptual** de por qué toda esa ingeniería de verificación/DLQ/poison es necesaria — porque el LLM no entiende ni razona de forma fiable. El **poison message** del cap. 9 es la encarnación operativa del LLM como endpoint poco fiable que este capítulo justifica filosóficamente.
- [[02 - Embeddings The Language of AI]] — el **LLM como endpoint poco fiable**: este capítulo lleva esa intuición a su fundamento (stochastic parrot, hallucination como consecuencia del training objective, no del bug).
- [[04 - Building Your First RAG App]] · [[08 - Pattern-Guided Coding]] — **[[RAG]] como la microarquitectura que constriñe al LLM a un dominio grounded/verificable**: el capítulo la nombra explícitamente como el deployment pattern más maduro, cerrando el arco que esos capítulos construyeron production-grade.
- [[01 - Introduction Patterns, Abstractions, and the GenAI Landscape]] — las **microarquitecturas** (RAG entre ellas) y la intuición del LLM como motor de probabilidad del próximo token, que este capítulo lleva a su conclusión epistemológica.
- [[07 - Tips and Best Practices]] — la **cabeza fría** frente a la "irrational/frothy exuberance" y el **[[Vendor Lock-in|vendor lock-in]]**; este capítulo da el sustento teórico de por qué el hype es peligroso (5 costos de ingeniería).
- **[[Stochastic Parrot]]** (Bender et al. 2021) · **[[Chinese Room]]** (Searle) · **[[Gödel incompleteness]]** (1931) / Penrose · **[[Universal Grammar]]** (Chomsky) — los 4 argumentos que delimitan lo que los LLMs no son (candidatos a nota propia).
- **[[Symbol Grounding Problem]]** (Harnad 1990) · **[[Hallucination]]** · **[[Computabilidad]]** · **[[Attention Mechanism]]** — los conceptos del reasoning gap (candidatos a nota propia).
- Pensadores invocados: **[[Chomsky]]** · **[[Roger Penrose]]** · **[[John Searle]]** · **[[Marvin Minsky]]** · **[[Alan Turing]]** · **[[Kurt Gödel]]** · **[[David Chalmers]]** — y los thought leaders contemporáneos (Hinton, Musk, Bengio, Ng) cuyas opiniones de AGI el capítulo pone en cuestión.
- **[[AGI]]** — Artificial General Intelligence: el claim central que el capítulo somete a "extraordinary claims require extraordinary evidence" (candidato a nota propia).
