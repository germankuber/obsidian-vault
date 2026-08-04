---
title: 06 - Evaluation and Testing
libro: Ontology Engineering
autor: Elisa F. Kendall / Deborah L. McGuinness
capitulo: 6
created: 2026-08-03
tags:
  - libros/ontology-engineering
  - type/reading-note
  - status/done
aliases:
  - Evaluation and Testing
  - Cap 6 - Evaluación y testing
updated: 2026-08-04
---

# 06 - Evaluation and Testing

> [!info] Capítulo 6 · *Ontology Engineering* — Elisa F. Kendall & Deborah L. McGuinness · Morgan & Claypool, *Synthesis Lectures on Data, Semantics, and Knowledge* (2019)
> Cómo saber si la ontología funciona: los dos ejes de **verificación** (¿está bien construida?) y **validación** (¿modela el dominio correcto?), el uso del razonador para detectar inconsistencias y **clases insatisfacibles**, las [[competency questions]] convertidas en tests ejecutables con [[SPARQL]], las dimensiones de calidad, y cómo montar esto como **integración continua** para una ontología. Navegá: [[_Ontology Engineering|Ontology Engineering]] · anterior [[05 - Ontology Design Patterns and Reuse]] · siguiente [[07 - Lifecycle, Versioning and Governance]].

## Resumen

Si la ontología es un artefacto de software, tiene que poder testearse — y el capítulo desarrolla exactamente eso. La distinción que organiza todo, ya anticipada en [[02 - Ontology Development Methodology]], es entre **verificación** y **validación**: la primera pregunta si la ontología está bien construida (consistente, sin clases insatisfacibles, con nomenclatura coherente); la segunda pregunta si modela el dominio correcto (responde las competency questions, el experto acepta las inferencias). Son ejes independientes, requieren herramientas distintas y los resuelven personas distintas. Una ontología puede ser lógicamente impecable y describir un dominio que no existe.

La verificación se apoya en el **razonador**, que es la herramienta distintiva del campo y no tiene equivalente exacto en el desarrollo de software convencional. El razonador computa la clasificación inferida, detecta inconsistencias y —sobre todo— señala **clases insatisfacibles**: clases que por sus axiomas no pueden tener ninguna instancia. El capítulo insiste en que una clase insatisfacible es **siempre** un error de modelado, nunca una decisión de diseño, y siempre bloqueante.

La validación se apoya en las **competency questions traducidas a consultas [[SPARQL]]**, ejecutadas contra la ontología poblada con datos reales. Ese es el equivalente a una suite de tests de aceptación: ejecutable, repetible, y con un criterio binario de éxito. El capítulo agrega el segundo mecanismo de validación, que ninguna herramienta reemplaza: llevarle al experto de dominio las **inferencias** del razonador —no los axiomas— y preguntarle si las clasificaciones deducidas son correctas.

Cierra con las dimensiones de calidad que van más allá de "funciona" —cobertura, precisión, claridad, concisión, mantenibilidad— y con el planteo de montar todo esto como **integración continua**: razonador y consultas corriendo en cada cambio, igual que un pipeline de tests.

## Verificación vs validación

Los dos ejes de la evaluación, que el capítulo trata como estrictamente independientes:

- **Verificación** — *¿está bien construida?* Es una propiedad **interna** del artefacto: coherencia lógica, ausencia de contradicciones, convenciones respetadas. Se comprueba con herramientas, principalmente el razonador. La resuelve el ontologista.
- **Validación** — *¿modela el dominio correcto?* Es una propiedad **externa**, relativa al mundo y al caso de uso. Se comprueba con competency questions y con el juicio del experto. La resuelve el experto de dominio.

> [!warning] **El error de proceso más frecuente es confundirlos.** El ontologista corre el razonador, ve que no hay inconsistencias, y declara la ontología terminada — sin que ningún experto haya revisado una sola inferencia. Verificación en verde no dice absolutamente nada sobre si el modelo describe el dominio real.

> [!note] La asimetría importante: **la verificación se puede automatizar casi por completo; la validación no.** Se puede automatizar la *ejecución* de las competency questions, pero decidir si el resultado es el correcto sigue requiriendo a alguien que conozca el dominio.

## El razonador como herramienta de verificación

El **razonador** (Pellet, HermiT, ELK, entre los citados habitualmente) es la herramienta distintiva del campo: computa las consecuencias lógicas de los axiomas declarados. Lo que aporta a la verificación:

- **Chequeo de consistencia** — ¿el conjunto de axiomas es contradictorio? Una ontología inconsistente es inservible: de una contradicción se sigue cualquier cosa, y el razonador puede "inferir" literalmente todo.
- **Clases insatisfacibles** — clases que no pueden tener ninguna instancia. El síntoma más útil del razonador.
- **Jerarquía inferida** — la taxonomía que el razonador deduce, que puede diferir de la declarada. Comparar ambas es un ejercicio de revisión enormemente productivo.
- **Clasificación de individuos** — a qué clases pertenece cada individuo según los axiomas, incluidas las que nadie declaró.

> [!warning] **Una clase insatisfacible es siempre un error, nunca una decisión.** Las causas típicas: disjunción declarada a la ligera entre clases que el modelo después combina, restricciones de cardinalidad que se contradicen, o un `range` que choca con una restricción de la clase. Nunca es cosmético y nunca conviene postergarlo — una insatisfacible temprana suele propagarse a decenas de subclases.

> [!tip] **La jerarquía inferida es el mejor instrumento de revisión que tenés.** Compará lo que declaraste con lo que el razonador dedujo: si aparecen relaciones de subclase que no esperabas, o dos clases que resultan equivalentes sin que lo hayas querido, encontraste un error de modelado que ninguna inspección visual del archivo habría mostrado.

> [!note] El costo de razonamiento crece con la expresividad usada. Los **perfiles de [[OWL]] 2** (EL, QL, RL) existen precisamente para esto: acotan la expresividad a cambio de garantías de complejidad. Si tu ontología tarda horas en clasificar, la pregunta correcta no es qué razonador usar sino **qué expresividad estás usando y si la necesitás**.

## Competency questions como tests ejecutables

El mecanismo de validación central, y la razón por la que [[02 - Ontology Development Methodology]] insiste en escribirlas antes de modelar.

El procedimiento:

1. **Traducir cada competency question a [[SPARQL]]** — la pregunta en lenguaje natural se vuelve una consulta ejecutable.
2. **Poblar la ontología con datos de prueba** representativos del dominio, incluidos casos borde.
3. **Definir el resultado esperado** — qué debería devolver la consulta para esos datos.
4. **Ejecutar y comparar** — igual que un test unitario.

```sparql
# CQ-07: ¿Qué medicamentos están contraindicados para un paciente con insuficiencia renal?
SELECT ?medicamento WHERE {
  ?medicamento  a                    :Medicamento ;
                :contraindicadoEn    ?condicion .
  ?condicion    rdfs:subClassOf*     :InsuficienciaRenal .
}
```

> [!note] **Este es el paralelo exacto con [[Test-Driven Development]]**: las competency questions se escriben antes del modelo, definen qué significa "terminado", y se ejecutan en cada cambio para detectar regresiones. La diferencia es que acá el "código" es un conjunto de axiomas y el "runtime" es un razonador.

> [!tip] Incluí en los datos de prueba los **casos borde que discutiste con el experto** durante la elicitación. Son exactamente los que revelan si el modelo captó la distinción o la perdió.

> [!warning] Una competency question que no se puede traducir a consulta ejecutable **no está lista** — es demasiado vaga. Si no podés escribir el SPARQL, tampoco vas a poder decidir si la ontología la responde, y perdiste el criterio de terminación que era su razón de ser.

### El tercer mecanismo: validar los datos con SHACL

Todo el libro repite *"OWL infiere, [[SHACL]] valida"* sin mostrar nunca una shape. Es el complemento operativo de la mitad de las advertencias sobre mundo abierto del capítulo 4:

```turtle
@prefix sh:  <http://www.w3.org/ns/shacl#> .
@prefix xsd: <http://www.w3.org/2001/XMLSchema#> .

:MedicamentoShape  a  sh:NodeShape ;
    sh:targetClass  :Medicamento ;

    sh:property [
        sh:path      :principioActivo ;
        sh:minCount  1 ;                    # ← OBLIGATORIO. OWL no puede exigir esto.
        sh:message   "Todo medicamento debe declarar al menos un principio activo"@es ;
    ] ;
    sh:property [
        sh:path          :dosisMaxima ;
        sh:datatype      xsd:decimal ;
        sh:minExclusive  0 ;
        sh:message       "La dosis máxima debe ser un decimal positivo"@es ;
    ] .
```

> [!note] **La diferencia con OWL, en una línea**: `sh:minCount 1` **falla** si el dato no está; `owl:minCardinality 1` **infiere** que existe aunque nadie lo haya declarado. No compiten — cubren necesidades distintas, y la mayoría de los sistemas productivos usan ambos: OWL para el modelo conceptual y la inferencia, SHACL para validar los datos que entran.

> [!tip] En un pipeline real, SHACL corre sobre los **datos** en el punto de ingesta, mientras el razonador corre sobre el **modelo** en CI. Son dos controles distintos en dos momentos distintos. Implementaciones: pySHACL (Python), TopBraid SHACL API (Java), y soporte nativo en GraphDB, Stardog y Jena.

### Tabla 6.1 — Los dos ejes de evaluación

| | Verificación | Validación |
|---|---|---|
| **Pregunta** | ¿Está bien construida? | ¿Modela el dominio correcto? |
| **Naturaleza** | Interna al artefacto | Externa, relativa al mundo |
| **Herramienta** | Razonador | Competency questions + juicio experto |
| **Señales de falla** | Inconsistencia, clases insatisfacibles | Consulta que devuelve resultados incorrectos o vacíos |
| **Quién la resuelve** | Ontologista | Experto de dominio |
| **¿Automatizable?** | Sí, casi por completo | Ejecución sí; juicio no |

> [!warning] La última fila es la trampa: automatizar la ejecución de las competency questions da sensación de cobertura completa, pero **alguien con conocimiento del dominio tiene que haber definido el resultado esperado**. Un test que verifica lo que el modelo ya hace no valida nada.

## Validar consecuencias, no afirmaciones

El segundo mecanismo de validación, y el que ninguna herramienta reemplaza. Ya planteado en [[02 - Ontology Development Methodology]], acá se vuelve procedimiento:

1. **Poblar** la ontología con instancias reales del dominio.
2. **Correr el razonador** para obtener clasificaciones e inferencias.
3. **Llevarle al experto las inferencias**, parafraseadas en lenguaje natural: *"el sistema dedujo que este caso es de tipo X — ¿es correcto?"*
4. **Investigar cada desacuerdo** hasta el axioma que lo causó.

> [!warning] **Un conjunto de axiomas individualmente correctos puede tener consecuencias que el experto jamás habría aceptado.** El experto revisa lo que escribiste y dice "sí, es correcto" — pero nadie le mostró que la combinación de dos axiomas inocentes prohíbe una categoría de casos que en su dominio existen. La divergencia entre modelo y mundo se produce en las consecuencias, no en las afirmaciones.

> [!tip] Presentá las inferencias en el lenguaje del experto, nunca como axiomas. *"Todo paciente con estas dos condiciones queda clasificado como de alto riesgo"* es revisable por un médico; el axioma [[OWL]] equivalente no lo es.

## Dimensiones de calidad

Más allá de "funciona", el capítulo enumera las dimensiones contra las que se evalúa una ontología:

- **Cobertura** — ¿cubre el alcance definido? ¿Responde todas las competency questions de la iteración?
- **Precisión** — ¿las definiciones discriminan bien? ¿Clasifica correctamente los casos borde?
- **Consistencia** — ¿es lógicamente coherente? Verificable con razonador.
- **Concisión** — ¿hay redundancia, clases duplicadas, axiomas que se implican entre sí? El exceso también es un defecto: cuesta mantener y enlentece el razonamiento.
- **Claridad** — ¿las definiciones son comprensibles? ¿La nomenclatura es coherente? ¿Está documentada?
- **Mantenibilidad** — ¿se puede extender sin romper lo existente? ¿Está modularizada?
- **Adecuación computacional** — ¿el razonamiento termina en tiempo aceptable para el caso de uso?

> [!note] Estas dimensiones **compiten entre sí**. Más cobertura suele costar concisión; más precisión suele costar adecuación computacional (más axiomas, razonamiento más lento). No hay un óptimo global — hay un balance que depende del caso de uso, exactamente como en cualquier decisión de arquitectura de software.

> [!tip] La dimensión más olvidada es la **adecuación computacional**. Una ontología preciosa que tarda cuarenta minutos en clasificar no sirve para un sistema que necesita responder en línea. Medí el tiempo de razonamiento desde temprano, no cuando ya es tarde para simplificar.

## Integración continua para ontologías

El capítulo cierra llevando la tesis del libro a su conclusión operativa: si es software, se integra continuamente. El pipeline mínimo:

1. **Ontología en control de versiones** — texto plano, diffeable. Formatos como Turtle o Manchester son mucho más legibles en diff que RDF/XML.
2. **En cada commit: correr el razonador** — chequeo de consistencia y detección de clases insatisfacibles. Falla el build si aparece alguna.
3. **En cada commit: ejecutar las competency questions** contra el dataset de prueba, comparando con los resultados esperados.
4. **Chequeos de convención** — nomenclatura, IRIs, presencia de etiquetas y definiciones en todas las entidades nuevas.
5. **Reporte de diff semántico** — no solo qué líneas cambiaron, sino **qué inferencias cambiaron**. Es la información que realmente importa para revisar un cambio.

> [!tip] El **diff semántico** es lo que distingue una revisión de ontología de una revisión de código. Dos cambios de una línea pueden tener consecuencias inferenciales radicalmente distintas: agregar una disjunción puede invalidar cientos de individuos. Revisá el cambio en las inferencias, no solo en el texto.

> [!warning] Sin este pipeline, los errores se descubren cuando alguien nota que una consulta devuelve basura — típicamente meses después del commit que lo causó, y con el contexto de esa decisión ya perdido.

### Con qué se implementa

El capítulo define los cinco pasos conceptualmente y no dice con qué herramienta. La respuesta —posterior al libro y hoy estándar de facto— es **ROBOT**, una herramienta de línea de comandos sobre la OWL API:

```bash
robot convert --input onto.owl --format ttl --output onto.ttl   # 1. normalizar serialización
robot reason  --input onto.ttl --reasoner ELK                   # 2. falla si hay insatisfacibles
robot report  --input onto.ttl --fail-on ERROR                  # 4. convenciones: labels, IRIs
robot verify  --input onto.ttl --queries tests/*.rq             # 3. competency questions
robot diff    --left main.ttl --right onto.ttl                  # 5. diff semántico
```

| Paso del capítulo | Comando |
|---|---|
| Control de versiones con formato diffeable | `robot convert` a Turtle, con orden fijo |
| Correr el razonador, fallar si hay insatisfacibles | `robot reason` |
| Ejecutar las competency questions | `robot verify` con las CQ como archivos `.rq` |
| Chequeos de convención | `robot report` |
| **Diff semántico** | `robot diff` — qué axiomas cambiaron, no qué líneas |

> [!tip] **`robot verify` es lo que convierte las competency questions en tests reales.** Escribís la CQ como SPARQL, definís qué no debe devolver, y el build falla cuando el modelo deja de responderla. Es la pieza que la teoría siempre pidió y que nadie sabía cómo implementar.

> [!note] Para proyectos con más de un autor y más de un release, el **Ontology Development Kit (ODK)** inicializa el repositorio entero —estructura, `Makefile` de release, CI configurado— con ROBOT adentro. Ver [[Herramental de ontologías]].

## Para aplicar

- **Separá verificación de validación** en tu proceso, con responsables distintos: razonador para una, experto para la otra.
- **Tratá toda clase insatisfacible como bloqueante** — nunca como deuda técnica a postergar.
- **Compará jerarquía declarada vs inferida** en cada iteración: las diferencias inesperadas son errores de modelado.
- **Traducí cada competency question a [[SPARQL]]** y mantené un dataset de prueba con casos borde reales.
- **Validá con el experto mostrándole inferencias parafraseadas**, nunca axiomas.
- **Medí el tiempo de razonamiento desde temprano** y revisá el perfil OWL si crece más de lo aceptable.
- **Montá integración continua**: razonador + competency questions en cada commit, con reporte de diff semántico.

## Conexiones

- [[_Ontology Engineering|Ontology Engineering]] — el MOC del libro.
- [[05 - Ontology Design Patterns and Reuse]] — capítulo anterior: los patrones cuyo resultado acá se evalúa · [[07 - Lifecycle, Versioning and Governance]] — capítulo siguiente: qué pasa con la ontología después de que pasa los tests.
- [[competency questions]] — el instrumento que acá se vuelve suite de tests ejecutable.
- [[SPARQL]] — el lenguaje de consulta que las hace ejecutables; **candidato a nota propia**.
- [[OWL]] — los perfiles EL/QL/RL como palanca de adecuación computacional.
- [[SHACL]] — complemento de validación estructural que OWL no cubre por su semántica de mundo abierto.
- [[Test-Driven Development]] — el paralelo metodológico directo: tests antes que implementación, como criterio de terminación.
- [[02 - Ontology Development Methodology]] — donde se introdujo la distinción verificación/validación que este capítulo desarrolla.
