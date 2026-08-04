---
title: 01 - Introduction Patterns, Abstractions, and the GenAI Landscape
libro: Building Complex Multi-Agent Systems Using Pattern Prompting
autor: Packt
capitulo: 1
created: 2026-06-19
tags:
  - libros/building-complex-multi-agent-systems-using-pattern-prompting
  - type/case-study
  - status/permanent
aliases:
  - Introduction Patterns, Abstractions, and the GenAI Landscape
  - Cap 1 - Patterns and the GenAI Landscape
updated: 2026-07-23
---

# 01 - Introduction Patterns, Abstractions, and the GenAI Landscape

> [!info] Capítulo 1 · *Building Complex Multi-Agent Systems Using Pattern Prompting* — Packt (ISBN 9781806114290)
> Sienta la filosofía del libro: construir aplicaciones [[GenAI]] se hace mejor con las herramientas, abstracciones y [[Design Patterns|patrones]] de ingeniería de software ya probados, no reinventando la rueda. Tiende un **puente de vocabulario** entre términos GenAI y conceptos IT (Tabla 1.1), da una vista a 10.000 pies de [[LLM|LLMs]], [[Vector Database|vector databases]] y [[Embeddings]], explica intuitivamente cómo funciona un LLM (probabilidad del próximo token), y define la **[[Agentic AI|IA agéntica]]** como arquitectura basada en componentes sobre LLMs. Es el cimiento conceptual de todo el libro y debe leerse antes de cualquier otro capítulo. Navegá: [[_Building Complex Multi-Agent Systems Using Pattern Prompting|Building Complex Multi-Agent Systems]] · siguiente [[02 - Embeddings The Language of AI]].

## Resumen

El capítulo introduce la **tesis central del libro**: GenAI tiene mucho más en común con el software tradicional de lo que se suele sugerir, y construir aplicaciones GenAI **solo se hace bien usando herramientas, paradigmas y mejores prácticas de ingeniería ya existentes y probadas** — no tecnologías ni enfoques enteramente nuevos. Prominentes entre esas prácticas están el uso de **abstracciones** y **[[Design Patterns|patrones]]**. El autor sostiene que si alguna vez usaste un whiteboard, ya entendés intuitivamente abstracción y patrones.

El recorrido arranca con un **puente de vocabulario** (Tabla 1.1): para cada término GenAI de sonido novedoso existe un equivalente IT que los ingenieros usan hace años (*agentic pattern* → microarchitecture, *agent* → component, *tool* → adapter, etc.). El mundo GenAI tiende, al menos por ahora, a **reinventar todo como algo nuevo** en vez de extender el conocimiento existente, lo que vuelve confuso el aprendizaje; bajo el capó, las herramientas nuevas usan los mismos patrones de orchestration y collaboration de los sistemas enterprise, aunque todavía sin su madurez y fiabilidad.

Sigue con **cómo estudiar GenAI** (cinco sugerencias: take the middle path, fundamentals-first, leverage what you already know, endpoint challenges, don't lose the forest for the trees), apoyándose en los **Enterprise Integration Patterns** de Hohpe & Woolf y en los **patrones GoF**. El libro adopta deliberadamente el término **microarchitecture** en lugar de "agentic pattern" para distinguir patrones de software consolidados de los "agentic patterns" nuevos y aún no probados. Explica el uso de **abstracciones y patrones** (la necesidad de *harmonización* de vocabulario, Figura 1.1), **cómo se usan los patrones en el libro** ([[Orchestration|orchestration]] vs [[Choreography|choreography]], soportados por [[RabbitMQ]]), y los **beneficios de descomponer microarquitecturas GenAI** en sus patrones constituyentes (soluciones a medida, extender herramientas enterprise, aprender herramientas nuevas rápido).

Luego construye intuición sobre **qué es un LLM** (analogía de la zebra; el LLM como continuación del big data; el LLM como *RESTful endpoint mal comportado*) y **cómo funciona** (probabilidad del próximo token, analogía con los autocompletados de búsqueda de Google/Amazon, Figuras 1.2 y 1.3, con la nota de que sigue siendo en parte un misterio). Cierra definiendo la **[[Agentic AI|IA agéntica]]** como la aplicación de **arquitectura basada en componentes** a sistemas que usan LLMs, donde agentes especializados colaboran, soportados por message brokers como [[RabbitMQ]] y reverse proxies como **ESBs** — con un enfoque **technology- y language-agnostic**. El próximo capítulo aborda los **vector embeddings** y el [[RAG]].

## El puente de vocabulario: términos GenAI y sus equivalentes IT

En un mundo fragmentado donde múltiples tecnologías y proveedores compiten por dominar, los conceptos viejos a veces se pierden o se reinventan y los términos familiares se redefinen. **GenAI no es la excepción**: se ve, al menos por ahora, una tendencia a **reinventar todo como algo nuevo** en vez de extender el conocimiento y las mejores prácticas de software ya existentes. Esto vuelve confuso aprender GenAI.

La **Tabla 1.1** funciona como un *cheat sheet* para traducir términos GenAI específicos a los términos más familiares del IT tradicional. La clave del capítulo: para **cada concepto GenAI de sonido novedoso existe un equivalente consolidado** que los ingenieros de software usan hace años.

> [!note] Bajo el capó, las herramientas GenAI nuevas usan **los mismos patrones de orchestration y collaboration** que se usan en sistemas enterprise desde hace muchos años — pero todavía **no tienen la madurez ni la fiabilidad** de sus contrapartes más viejas.

Un **vocabulario harmonizado** ayuda además a entender mejor las similitudes entre las herramientas que usamos y a **encontrar la tecnología correcta para el trabajo**. El resto del capítulo es un survey del paisaje GenAI a 10.000 pies (vector databases, [[Embeddings|embeddings]] y [[LLM|LLMs]]): aunque high-level, cubre el conocimiento núcleo para construir aplicaciones GenAI y es el cimiento del libro entero.

### Tabla 1.1 — Cheat sheet de terminología: términos GenAI mapeados a equivalentes IT tradicionales

| GenAI Term | Traditional IT Equivalent |
|---|---|
| Agentic pattern | Microarchitecture composed of GoF and Fowler patterns |
| Agent | Component |
| Short-term memory | Session state |
| Long-term memory | System of record |
| Multi-agent system | Component architecture |
| Tool | Adapter |

> [!tip] No hay que reinventar la rueda solo porque trabajás con un servicio o API RESTful de un LLM: cada novedad GenAI tiene un patrón IT establecido detrás. Aprendé el equivalente IT y la novedad se vuelve familiar al instante.

## Estudiar GenAI

El conocimiento sobre GenAI **debería sentirse como una extensión natural** de la experiencia en ingeniería de software. **Orchestration y collaboration están en el corazón de GenAI**; cualquier trabajo previo con integraciones o sistemas de mensajería se transfiere directamente. Si trabajaste orquestando web services o ruteando mensajes, ya poseés habilidades clave — **message routing, back-office system integration y aplicación de integration patterns** — esenciales para construir aplicaciones GenAI.

Los **[[Enterprise Integration Patterns]]** de **Hohpe y Woolf** y los **patrones [[GoF|Gang of Four]]** (*Design Patterns: Elements of Reusable Object-Oriented Software*) sustentan todos los llamados "agentic patterns".

> [!note] Para preservar el significado de los patrones tal como los entendían **Fowler y los GoF**, el libro usa el término **microarchitecture** en lugar de "agentic pattern" a lo largo de todo el texto.

Las cinco sugerencias para aprender GenAI de forma efectiva:

- **Take the middle path (tomá el camino del medio)** — al empezar, conviene saltear los deep dives en matemática. Muchas preguntas profundas sobre IA todavía no tienen respuesta. De hecho, **la IA es el primer campo de la historia con dos ramas de investigación**: una hace descubrimientos nuevos y la otra intenta entender por qué esos descubrimientos funcionan. Nuestro trabajo como ingenieros es **usar la IA para resolver un problema, no descifrar por qué funciona**.
- **Fundamentals-first approach (fundamentos primero)** — aprender GenAI a través de una herramienta o producto específico tiene desventajas. Incluso un progreso incremental pequeño en los LLMs puede tener **efectos dominó mayores** que fragmentan aún más un mercado ya fragmentado de herramientas y microarquitecturas. Sin los fundamentos primero, cada enfoque más avanzado exigirá más tiempo y esfuerzo para absorberse.
- **Leverage what you already know (apalancá lo que ya sabés)** — asociar un tema nuevo con algo que ya conocés acelera enormemente el aprendizaje. Los **patrones son omnipresentes** en software, y GenAI no es la excepción: proveen un **modelo mental** que vuelve lo desconocido familiar casi al instante. No hace falta reinventar la rueda solo por trabajar con un servicio o API RESTful de un LLM.
- **Be aware of endpoint challenges (ojo con los desafíos de endpoints)** — el reto de trabajar con endpoints de LLM es **integrarlos en entornos que demandan velocidad, fiabilidad, seguridad y escalabilidad** — rasgos que estos endpoints no siempre garantizan. Probablemente ya enfrentaste endpoints o APIs que se comportan mal; el mundo del software ya invirtió mucho tiempo resolviendo esos problemas y las soluciones se transfieren directamente.
- **Don't lose sight of the forest for the trees (no pierdas el bosque por los árboles)** — usar abstracciones evita ahogarse en detalle de bajo nivel excesivo, especialmente en tecnologías que cambian rápido (cambios significativos ocurren incluso antes de que el material de aprendizaje se finalice). Los ingenieros experimentados usan **el lente de la abstracción** para captar el conocimiento esencial filtrando el detalle distractor. Elegir herramientas es como apostar en una carrera de caballos: **elegí el caballo equivocado y volvés con los bolsillos vacíos**; hay muchos caballos en esta carrera. Lo prudente es aprender a través de abstracciones y patrones — no importa qué herramienta elijas después, la vas a aprender rápido y con un entendimiento más profundo si primero entendés los principios del libro.

> [!warning] Resistí la tentación de **atar tu carrera a un producto o framework**. El paisaje GenAI está demasiado fragmentado y evoluciona demasiado rápido; los principios duran, las herramientas específicas no.

## El uso de abstracciones y patrones

El enfoque del libro es aprender GenAI **examinando las microarquitecturas** usadas en aplicaciones GenAI y **descomponiéndolas en sus patrones y abstracciones constituyentes**. Como ingenieros, no queremos meternos en los rápidos de tecnologías que evolucionan rápido más de lo necesario, ni detenernos innecesariamente en los detalles de un producto que todavía cambia a gran velocidad.

Una de las razones más contundentes para usar abstracciones y [[Design Patterns|design patterns]] en GenAI es la necesidad de **harmonización** de GenAI con las arquitecturas IT y la documentación técnica existentes. La harmonización **requiere que todos usen el mismo vocabulario** al describir cosas similares.

> [!note] Cuando dos campos usan vocabularios distintos para describir los mismos conceptos, vuelven **doblemente difícil** el intercambio de ideas.

La **Figura 1.1** ilustra exactamente esta brecha: los mismos conceptos aparecen bajo nombres completamente distintos según qué comunidad esté hablando (expertos en IA vs. ingenieros de software).

![[B34134_1_1.png|286]]
*Figure 1.1 — The vocabulary gap between AI experts and software engineers. The same concepts appear under entirely different names depending on which community is speaking.*

## Cómo se usan los patrones en este libro

Los **design patterns** son soluciones generalizadas y reutilizables a problemas recurrentes del desarrollo de software. Se popularizaron por obras seminales como *Design Patterns: Elements of Reusable Object-Oriented Software* (el libro de la **"[[GoF|Gang of Four]]"**) y *[[Enterprise Integration Patterns]]*. **Estudiar patrones es un atajo** para aprender cualquier tecnología o herramienta nueva: la mayoría de la documentación de developers usa diagramas construidos alrededor de patrones y otras abstracciones.

Pensar en patrones es particularmente aplicable a GenAI porque este se apoya en **dos conceptos clave** de ingeniería de software que dependen fuertemente de los design patterns:

- **[[Orchestration|Orchestration]]** — un componente central publica un evento que dispara acciones en otros componentes. El **flujo de control se coordina de forma centralizada**.
- **[[Choreography|Choreography]]** — no hay controlador central; cada componente **sabe cuándo actuar** según reglas e interacciones predefinidas, permitiendo que el sistema se coordine de forma **descentralizada y auto-organizada**.

Ambos paradigmas son comunes en software enterprise y están **bien soportados por [[RabbitMQ]]**, un software message-broker que actúa como middleware para facilitar la comunicación entre servicios.

> [!tip] No hace falta conocimiento previo de estas herramientas para completar los ejercicios del libro, y su uso **no debe verse como una recomendación**. La razón principal de elegirlas es que soportan muchos lenguajes de programación y por tanto pueden ser entendidas y extendidas por ingenieros usando los lenguajes que ya conocen.

## Beneficios de descomponer las microarquitecturas GenAI en sus patrones constituyentes

Buena parte de aprender GenAI es aprender su **microarchitecture**. El libro usa el término *microarchitecture* en vez de "agentic pattern" para marcar una **distinción clara** entre patrones de software consolidados y los nuevos "agentic patterns" aún no probados. El enfoque para entender microarquitecturas es **descomponerlas en patrones y abstracciones**.

Descomponer las microarquitecturas GenAI en sus patrones constituyentes ofrece múltiples beneficios:

- **Permite construir soluciones a medida (custom solutions)** — GenAI tiene muchos "agentic patterns" (microarquitecturas) y un número creciente de herramientas que orquestan y coreografían la colaboración entre componentes GenAI y otras partes de IT. Pero estas microarquitecturas y frameworks suelen ser **demasiado coarse-grained** para los requisitos complejos del mundo real, que con frecuencia demandan soluciones únicas. El **capacity planning** para implementarlas también requiere entender cómo funcionan por dentro. Descomponiéndolas en patrones familiares, los ingenieros pueden razonar con confianza sobre **performance, scalability y failure modes**.
- **Permite extender tus herramientas enterprise para casos GenAI** — una vez que entendemos los patrones usados en GenAI, podemos **identificar herramientas y librerías maduras** que ya los soportan. Muchas de esas herramientas son confiables y ya se usan en empresas para conectarse a sistemas back-office; **la confianza que ganaron** de ingenieros y líderes tecnológicos hace la adopción e integración mucho más suaves.
- **Habilita el aprendizaje rápido de herramientas y productos nuevos** — tras aprender los fundamentos a través de patrones, aprender nuevas herramientas se vuelve fácil. Mantener el conocimiento actualizado en un mundo GenAI cada vez más fragmentado exige **mucho menos esfuerzo cuando el cimiento son principios** y no implementaciones específicas.

> [!note] Para construir aplicaciones GenAI de forma inteligente hay que tener un **entendimiento high-level y buena intuición** de cómo funcionan los LLMs. **El uso de un LLM es lo que define a GenAI.** El libro saltea la matemática y el deep learning, y se enfoca en ejemplos que construyen intuición.

## Qué es un LLM

La pregunta "qué es algo" puede tener muchas respuestas válidas. Por ejemplo, ante **"¿qué es una zebra?"** podríamos recibir:

- Un animal rayado que parece un caballo.
- Un miembro del género **Equus**.
- Un pariente del caballo que vive en las llanuras de África, mide entre **3.81 y 4.79 pies** a la altura del hombro y pesa entre **450 y 948 libras**.

Ninguna respuesta es incorrecta; cada una puede ser la mejor en ciertas circunstancias. La moraleja: **encontrar la respuesta correcta suele implicar buscar entre muchas respuestas correctas la más útil para un contexto particular**. Como ingenieros, debemos encontrar la respuesta a "¿qué es un LLM?" que mejor sirva para **construir aplicaciones LLM**.

Un **[[LLM]]** puede describirse de muchas formas, entre ellas:

- **Como la próxima generación de procesamiento de texto después del Big Data** — esta perspectiva muestra cómo se comportan los LLMs *funcionalmente*.
- **Como un RESTful endpoint** que es no-determinista, no-performante, no fiable, inseguro y a menudo con alta latencia — esta perspectiva guía cuando resolvemos *desafíos de infraestructura*.

### El LLM como continuación del big data

Los datos han sido llamados *"the new oil"*: el **90% de los datos del mundo se produjo en solo los últimos dos años**, y se generan unos asombrosos **402.74 millones de terabytes de datos cada día**. Durante décadas se invirtieron miles de millones de dólares en extraer valor de los datos, creando multitud de tecnologías nuevas. Muchas empresas creyeron, con razón o no, que los datos eran un *"magic bullet"* para entender profundamente a sus clientes y hallar nuevas formas de marketing.

Pese a tener solo herramientas primitivas comparadas con los LLMs y trabajar con cantidades de datos comparativamente pequeñas, las empresas gastaban miles de millones por año para cosechar pequeños insights. A menudo creaban **data lakes** agregando petabytes de datos propios combinados con datos comprados (a veces a gran costo), y luego usaban **data warehouses, business analytics platforms, data mining software y cloud analytics services** para extraer insights.

> [!note] Los LLMs pueden pensarse como la **próxima generación** de esas herramientas — pero están tan adelantados que uno duda en ponerlos en la misma categoría. A diferencia de sus predecesores, **dan la impresión convincente de entender los datos**, pueden entrenarse en casi toda la internet **comprimiendo conocimiento en unos pocos miles de millones de parámetros**, y responden a preguntas del usuario (los **prompts**), reemplazando dashboards y analytics con queries en lenguaje natural.

Podría decirse correctamente que **los LLMs entregan el valor que se prometía del big data, y algo más**.

### El LLM como un RESTful endpoint mal comportado

En muchos de los desafíos que enfrentarás —asegurar que tu aplicación sea segura, escalable y fiable— el hecho de que el endpoint que llamás sea un LLM es **a menudo irrelevante**. Su comportamiento es esencialmente el mismo que el de cualquier web service con las mismas características: podés usar **las mismas técnicas y herramientas** que ya usás con endpoints no-LLM.

> [!warning] Escribir código que funciona **no significa que la tarea esté terminada**. La mayoría de las aplicaciones deben implementar **logging, messaging, parallel execution, asynchronous callouts, bottleneck management, error handling, security, failover y redundancy**. Implementar una aplicación GenAI no es la excepción si querés un producto robusto.

Usar los patrones de software correctos también resuelve varios de estos desafíos. En los capítulos siguientes el libro recorre varios casos y muestra cómo superarlos usando **advanced messaging** y tecnologías de **containers y orchestration** que dan soluciones drop-in a problemas comunes de ingeniería GenAI. LLMs como **Grok, ChatGPT y Gemini** ya son parte de la vida cotidiana; aunque GenAI se apoya en matemática avanzada y deep learning, su núcleo se construye sobre unos pocos conceptos high-level.

## Cómo funcionan los LLMs

Para entender cómo funcionan los LLMs por dentro, el libro construye una **idealized view**: usa la abstracción para saltear los detalles de bajo nivel y que las grandes ideas afloren con claridad. **Idealizar es una técnica bien aceptada** usada por científicos, incluidos los físicos teóricos, al explorar temas muy complejos — idealizar un tema es simplemente otra forma de aplicar abstracción.

> [!note] La pieza esencial de conocimiento: **los LLMs usan probabilidad para encontrar la próxima palabra o conjunto de palabras** en una oración. **No hay lógica, no hay if-else.** Los LLMs simplemente escanean miles de millones de oraciones y encuentran las completaciones más probables.

Podemos desarrollar intuición con un pequeño experimento usando los **autocompletados del search box de Google o Amazon**: igual que los LLMs, los search boxes operan **probabilísticamente** para hallar la próxima palabra más probable dado un string. Por ejemplo, al teclear **"I want to buy"** aparece un dropdown con sugerencias basadas en el historial de búsqueda, como *"I want to buy a couple of magazines"*, *"I want to buy a car one you have one"* y *"I want to buy a new jacket"*.

![[B34134_1_2.png|479]]
*Figure 1.2 — Search completions: typing a partial phrase returns probabilistic suggestions, analogous to how an LLM predicts its next token.*

Si seleccionás una opción, por ejemplo *"I want to buy a new jacket"*, se presenta una **nueva lista de completaciones**, cada una reflejando la continuación estadísticamente más probable dadas las palabras elegidas hasta el momento.

![[B34134_1_3.png|526]]
*Figure 1.3 — Updated sentence completions: selecting a suggestion refines the probabilistic predictions based on the full context entered so far.*

Continuando el proceso de seleccionar sugerencias se llega a una oración completa. Se puede inferir cómo Google, Amazon y otros derivan los completados de query: sus ingenieros probablemente construyeron un sistema que **registra queries previas y cuenta cuántas veces se tecleó cada una**, devolviendo las oraciones con mayor cantidad de ocurrencias (las de mayor probabilidad de coincidir con la intención del usuario). Ahora imaginá que la lista de oraciones se derivara no solo de una DB de queries previas sino de **casi toda la internet**: eso es, a alto nivel, lo que hacen los LLMs, y por eso pueden **completar estrofas de poesía, escribir reportes de investigación, responder preguntas de matemática y mucho más**.

> [!note] **¿Por qué sigue siendo en gran parte un misterio?** Aunque esta explicación parezca incompleta o confusa, no hay que preocuparse: **nadie la entiende del todo**. La IA tiene una rama entera de investigación dedicada a entender por qué se logran sus resultados. La **incapacidad de explicar plenamente los resultados es una característica fundamental del campo**, no un hueco del libro.

## Qué es la IA agéntica

> [!note] **[[Agentic AI|Agentic AI]] es la aplicación de arquitectura basada en componentes (component-based architecture) a sistemas que usan LLMs.** Las ventajas de las arquitecturas de componentes sobre las monolíticas se reconocen como best practice desde hace décadas.

Un **AI agent**, el bloque fundamental de las aplicaciones agénticas, puede entrenarse y ser competente en distintos tipos de tareas. Por ejemplo, un LLM puede ser bueno **razonando** mientras otro es mejor **resumiendo** grandes volúmenes de texto. Uno empieza a preguntarse cómo colaborarían estos "agentes": quizás el **reasoning agent** halla soluciones y pasa el resultado al **summarization agent** para comprimirlo; o quizás el summarization agent primero comprime megabytes de texto antes de pasarlo al reasoning agent.

Esta **colaboración entre agentes especializados** es un foco mayor de GenAI. Hay **formas específicas en que los agentes pueden interactuar** que se consideran particularmente útiles, mientras que otras colaboraciones causan problemas; el libro estudiará esos patrones en capítulos posteriores junto con las herramientas que los implementan — herramientas confiables tanto para líderes IT como para ingenieros.

> [!tip] Hoy hay muchos productos para construir aplicaciones agénticas, algunos muy buenos. Pero **no necesitás herramientas agénticas para construir una aplicación agéntica**. El libro adopta un enfoque **technology- y language-agnostic**: no enseña herramientas ni productos. Para los ejercicios de código podés usar tu lenguaje preferido para extender los frameworks que se proveerán.

También se puede explorar construir las propias microarquitecturas usando **advanced messaging** soportado por message brokers como **[[RabbitMQ]]** y reverse proxies como los **enterprise service buses (ESBs)**, que tienen bindings para casi todo lenguaje popular. Construir aplicaciones agénticas con **herramientas maduras y bien probadas es probablemente el mejor enfoque**, sobre todo si vas a desplegar en una empresa grande o esperás servir a muchos usuarios. El foco del libro es **entender los principios y patrones subyacentes**, no aprender implementaciones específicas: completando los ejercicios en un lenguaje que ya conocés, desarrollarás un entendimiento claro de cómo tomar las mejores decisiones de desarrollo para lo que sea que estés construyendo.

## Citas

> "Building GenAI applications can only be done well by using existing tools, paradigms, and long-standing engineering best practices."

> "Software patterns have thus become the lingua franca of vibe coding as well."

> "Notice that for every new-sounding GenAI concept there is an established equivalent that software engineers have been using for years."

> "AI is the first field in history that has two branches of research: one branch makes new discoveries while the other branch tries to figure out why those discoveries work."

> "Picking tools is like betting in a horse race: pick the wrong horse and go home with empty pockets. There are many horses in this race."

> "There is no logic, no if-else statements. LLMs are simply scanning across billions of sentences and finding the most likely completions."

> "One could correctly say that LLMs are delivering the value that was promised from big data, and then some."

> "GenAI has introduced new microarchitectures — but not new fundamental software patterns."

## Para aplicar

- **Traducí cada término GenAI a su equivalente IT** antes de adoptar una herramienta — usá la Tabla 1.1 como cheat sheet para no aprender de cero lo que ya conocés (agent→component, tool→adapter, multi-agent system→component architecture).
- **Aprendé fundamentos primero, no herramientas** — invertí en abstracciones y patrones (GoF, Hohpe & Woolf) y no en un producto/framework concreto; así absorbés cualquier herramienta nueva rápido y con entendimiento profundo.
- **Tratá al endpoint LLM como cualquier web service mal comportado** — aplicá las mismas técnicas que ya usás: logging, messaging, parallel execution, async callouts, bottleneck management, error handling, security, failover y redundancy.
- **Descomponé toda microarquitectura GenAI en patrones constituyentes** para razonar con confianza sobre performance, scalability y failure modes, e identificar herramientas maduras que ya los soporten.
- **Decidí orchestration vs choreography** según necesites control centralizado (un componente coordina) o coordinación descentralizada auto-organizada; ambos están soportados por message brokers como RabbitMQ.
- **Construí agéntico de forma technology-agnostic** — podés armar tus propias microarquitecturas con RabbitMQ y ESBs (bindings para casi todo lenguaje), sin atarte a un framework agéntico específico.
- **No saltees este capítulo** — el resto del libro asume haberlo leído y entendido.

## Conexiones

- [[_Building Complex Multi-Agent Systems Using Pattern Prompting|Libro]] — el MOC del libro.
- [[02 - Embeddings The Language of AI]] — capítulo siguiente: qué son los **vector embeddings**, cómo representan significado como números y por qué son centrales al **semantic search**, **similarity matching** y **[[RAG|retrieval-augmented generation]]**.
- [[Design Patterns]] · **[[GoF|Gang of Four]]** · **[[Enterprise Integration Patterns]]** (Hohpe & Woolf) — los patrones que sustentan toda microarquitectura GenAI.
- [[Orchestration]] · [[Choreography]] — los dos paradigmas de coordinación; soportados por [[RabbitMQ]] y **ESBs**.
- [[LLM]] · [[Embeddings]] · [[Vector Database]] — el survey high-level del paisaje GenAI.
- [[Agentic AI]] — IA agéntica como arquitectura basada en componentes sobre LLMs (candidato a nota propia transversal del libro).
- **[[Microarchitecture]]** — el término que el libro usa en lugar de "agentic pattern" (candidato a nota propia).
