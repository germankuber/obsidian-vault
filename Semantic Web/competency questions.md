---
title: competency questions
source: (conocimiento general del campo — Grüninger & Fox 1995; Kendall & McGuinness 2019. Verificar antes de citar)
author: —
created: 2026-08-03
tags:
  - semantic-web
  - type/concept
  - status/stub
aliases:
  - competency questions
  - competency question
  - Competency Questions
  - preguntas de competencia
  - CQ
---

# competency questions

> [!note] Definición
> Una **competency question** es una pregunta concreta, escrita en lenguaje natural y en el vocabulario del negocio, que la [[ontología]] debe poder responder una vez construida. No es una pregunta *sobre* la ontología ("¿cuántas clases tiene?") sino una pregunta **del dominio** que el sistema tendrá que contestar.

Ejemplos reales:

- *¿Qué medicamentos están contraindicados para un paciente con insuficiencia renal?*
- *¿Qué empleados tienen la certificación vigente para operar este equipo?*
- *¿Qué componentes de este producto vienen de proveedores de una región determinada?*
- *¿Qué pólizas cubren este tipo de siniestro en esta jurisdicción?*

Son simples de escribir y engañosamente potentes: **son el instrumento más transversal de la ingeniería de ontologías**, y la razón principal por la que un proyecto termina en vez de derivar para siempre.

## Por qué importan tanto: cumplen tres funciones a la vez

Es lo que las distingue de un documento de requisitos común. Un solo artefacto cubre tres necesidades que normalmente vivirían en tres documentos desincronizados entre sí:

| Función | Qué responde | Cuándo se usa | Qué pasa si falta |
|---|---|---|---|
| **Requisitos** | ¿Qué tiene que estar modelado? | Al inicio, antes de modelar | Se modela por intuición; aparecen conceptos que nadie pidió |
| **Alcance / frontera** | ¿Qué queda afuera? | Al negociar alcance con stakeholders | El alcance se desborda; el proyecto no termina nunca |
| **Test de aceptación** | ¿Está lista la ontología? | Al cerrar, como consulta ejecutable | No hay criterio objetivo de terminación |

**Como requisitos**: si una pregunta menciona *contraindicación*, *insuficiencia renal* y *medicamento*, esos tres conceptos y la relación entre ellos **tienen** que existir en el modelo. La pregunta dicta qué modelar, sin ambigüedad.

**Como alcance**: y su contracara, la frontera. Lo que **ninguna** competency question toca queda deliberadamente afuera. Este es el uso más subestimado y el más valioso políticamente — da un criterio objetivo para rechazar pedidos de modelado que nadie va a usar, sin que la discusión se vuelva una pulseada de opiniones.

**Como tests**: la ontología está lista cuando las responde todas. Y son **ejecutables**: se traducen a consultas [[SPARQL]] que corren contra la ontología poblada y verifican que devuelvan lo esperado.

> [!tip] Escribilas **antes de modelar nada**. Sin ellas el modelado no tiene fin natural —siempre hay un concepto más que agregar, una distinción más fina que hacer— y el proyecto deriva hacia el modelado por el modelado hasta morir por agotamiento presupuestario. Con ellas tenés a la vez el punto de partida y la línea de llegada.

## La virtud política: todos las entienden

> [!note] Se escriben en **lenguaje de negocio, no en lenguaje formal**. Un stakeholder que no sabe qué es una lógica descriptiva puede leerlas, discutirlas, priorizarlas y aprobarlas. Es el único artefacto del proyecto que **todos los roles comprenden sin traducción**.

Esto resuelve parcialmente el problema de traducción entre experto de dominio y ontologista que es el riesgo número uno de estos proyectos. El experto no puede validar un axioma [[OWL]]; sí puede validar *"¿el sistema tiene que poder responder esto?"*.

## De pregunta a test ejecutable

El ciclo completo, que es donde se ve por qué valen tanto:

1. **Escribir la pregunta** en lenguaje natural, con el experto de dominio.
2. **Identificar los conceptos** que menciona — esos son tus requisitos de modelado.
3. **Modelar** lo mínimo que la responda.
4. **Traducir a [[SPARQL]]** — la pregunta se vuelve consulta.
5. **Poblar** con datos de prueba, incluidos casos borde.
6. **Ejecutar y comparar** contra el resultado esperado. Igual que un test unitario.

```sparql
# CQ-07: ¿Qué medicamentos están contraindicados para un paciente con insuficiencia renal?
SELECT ?medicamento WHERE {
  ?medicamento a                 :Medicamento ;
               :contraindicadoEn ?condicion .
  ?condicion   rdfs:subClassOf*  :InsuficienciaRenal .
}
```

> [!tip] Para tests, **`ASK` suele ser mejor que `SELECT`**: devuelve un booleano, que es exactamente lo que un test necesita afirmar. `ASK { ... }` responde *"¿el grafo satisface esta condición?"* sin traer datos.

## Cómo escribir buenas competency questions

- **Concretas, no vagas.** *"¿Qué sabemos sobre nuestros clientes?"* no acota, no testea, no se traduce a consulta. Si no podés imaginar la respuesta como una lista concreta de resultados, la pregunta todavía no está lista.
- **En el vocabulario del negocio**, no en el del modelo. Usá las palabras que el experto usa.
- **Entre 5 y 15 por iteración.** Suficientes para cubrir el alcance de la vuelta, pocas como para modelarlas de verdad.
- **Priorizadas.** No todas valen lo mismo; las del caso de uso principal gobiernan las decisiones de diseño cuando hay tensión entre alternativas.
- **Con casos borde incluidos.** Las preguntas que rozan la frontera del concepto son las que revelan si el modelo captó la distinción o la perdió.
- **Verificables.** Si no podés escribir el SPARQL, tampoco vas a poder decidir si la ontología la responde — y perdiste el criterio de terminación que era su razón de ser.

> [!warning] **La prueba de fuego**: si no podés escribir la consulta que la responde, la competency question no está terminada. No es un problema de la herramienta ni de tu SPARQL — es que la pregunta todavía es demasiado vaga para guiar el modelado.

## Non-goals: la contracara imprescindible

Un artefacto barato y muy efectivo que acompaña a la lista: las **preguntas que explícitamente NO se van a responder**, cada una con una línea de justificación.

> [!tip] Es el documento al que se apunta cuando aparece el pedido fuera de alcance. Evita rediscutir la misma frontera cada trimestre y convierte una discusión política en una consulta a un documento acordado.

Sin non-goals declarados, las fronteras se desbordan solas: alguien pide modelar una factura, después un asiento contable, y el proyecto adquirió un segundo dominio sin que nadie lo haya decidido.

## Cómo resuelven las decisiones de modelado

Su alcance va más allá de los requisitos: son el **criterio que desempata** cuando una decisión de modelado tiene varias alternativas legítimas.

- **¿Clase o instancia?** *Golden Retriever* puede ser una clase de perros o una instancia de *Raza*. No hay respuesta correcta en abstracto — la da la pregunta que tenés que responder. Si necesitás *"¿cuántas razas registró el club en 2024?"*, es instancia.
- **¿Qué escalón del [[espectro semántico]]?** Si ninguna pregunta requiere inferencia, no necesitás [[OWL]]: [[SKOS]] alcanza y es mucho más barato de construir y mantener.
- **¿Modelar tiempo?** Solo si alguna pregunta es histórica (*"¿quién era el responsable en 2021?"*). Modelar tiempo multiplica la complejidad de todo el modelo.
- **¿Reusar esta ontología externa?** Evaluá la candidata contra tus propias competency questions: si no ayuda a responder ninguna, no la reuses por prestigio.
- **¿Hasta dónde bajar la jerarquía?** Hasta donde las preguntas lo exijan, no hasta agotar el dominio.

> [!note] Es el mismo criterio aplicado una y otra vez: **el caso de uso manda, no la ambición de completitud**. Las competency questions son la forma concreta que toma "el caso de uso" en cada decisión.

## El paralelo con TDD

> [!note] La analogía es exacta y vale la pena tenerla presente: **las competency questions son a la ontología lo que los tests al código**. Se escriben **antes**, definen qué significa "terminado", y se ejecutan en cada cambio para detectar regresiones. La diferencia es que acá el "código" es un conjunto de axiomas y el "runtime" es un razonador.

Como en [[Test-Driven Development]], el beneficio principal no es la detección de errores sino el **diseño**: escribir la pregunta primero te obliga a clarificar qué necesitás antes de construirlo.

## Errores comunes

- **Escribirlas después de modelar** — se vuelven una justificación del modelo que ya hiciste, no un requisito que lo guía. Pierden toda su función.
- **Demasiado vagas** — no acotan, no testean, no se traducen.
- **No ejecutarlas nunca** — quedan como documento decorativo. Si no corren en cada iteración, no son tests.
- **Sin non-goals** — solo la mitad del trabajo; el alcance se desborda igual.
- **Definir el resultado esperado sin el experto** — un test que verifica lo que el modelo ya hace no valida nada.
- **Confundirlas con preguntas sobre la ontología** — *"¿cuántas clases tiene?"* no es una competency question; es una métrica.

## Conexión en el vault

- Es el instrumento más transversal de [[_Ontology Engineering|Ontology Engineering]]: nacen en [[01 - Introduction]], se vuelven método en [[02 - Ontology Development Methodology]], resuelven dudas de análisis y modelado en [[03 - Terminology and Domain Analysis]] y [[04 - Modeling Decisions]], filtran el reuso en [[05 - Ontology Design Patterns and Reuse]], se ejecutan como tests en [[06 - Evaluation and Testing]] y protegen contra regresiones en [[07 - Lifecycle, Versioning and Governance]].
- Se vuelven ejecutables vía [[SPARQL]] — sin esa traducción son solo buenas intenciones.
- Deciden el escalón del [[espectro semántico]]: son el criterio que evita la sobre-formalización, que es una de las causas de fracaso de estos proyectos.
- Aplican igual a un [[Knowledge graph]] aunque nadie en el proyecto use la palabra "ontología": la ausencia del término no es ausencia del problema.

## References

- Grüninger, M. & Fox, M. (1995) — *Methodology for the Design and Evaluation of Ontologies*. El trabajo que introduce el término.
- Kendall, E. & McGuinness, D. (2019) — *Ontology Engineering*, Morgan & Claypool.
- [SPARQL 1.1 Query Language](https://www.w3.org/TR/sparql11-query/) — para traducirlas a tests ejecutables.

## Related

- [[SPARQL]]
- [[ontología]]
- [[espectro semántico]]
- [[OWL]]
- [[SKOS]]
- [[Test-Driven Development]]
- [[_Ontology Engineering|Ontology Engineering]]
