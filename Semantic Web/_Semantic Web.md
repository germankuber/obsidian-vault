---
title: Semantic Web — MOC
created: 2026-08-03
tags:
  - semantic-web
  - type/moc
  - status/done
aliases:
  - Semantic Web
  - Web Semántica
  - Semantic Web MOC
updated: 2026-08-04
---

# 🕸️ Semantic Web — MOC

> [!info] Punto de entrada
> El stack de tecnologías para representar **significado** de forma procesable por máquina: grafos de datos, vocabularios formales, razonamiento y consulta. Es la base técnica de las ontologías y los knowledge graphs.

## 🎯 De qué se trata

La idea central: en vez de que el significado de los datos viva **implícito** —en el código de la aplicación, en la cabeza de un desarrollador, en el nombre de una columna— se lo declara de forma **explícita y formal**, en un vocabulario compartido que cualquier sistema puede leer y sobre el que una máquina puede razonar.

De ahí sale un stack de capas acumulativas, donde cada una se apoya en la anterior:

| Capa | Estándar | Qué agrega |
|---|---|---|
| **Modelo de datos** | [[RDF]] | Todo es una tripleta sujeto-predicado-objeto: un grafo |
| **Esquema básico** | [[RDFS]] | Clases, propiedades, jerarquía (`subClassOf`, `domain`, `range`) |
| **Lógica formal** | [[OWL]] | Axiomas, restricciones, **razonamiento automático** |
| **Vocabularios ligeros** | [[SKOS]] | Taxonomías y thesauri sin exigir formalización lógica |
| **Consulta** | [[SPARQL]] | Pattern matching sobre el grafo + protocolo HTTP |
| **Validación** | [[SHACL]] | Verificar que los datos cumplan una forma esperada |

> [!note] **La confusión más común**: [[OWL]] y [[SHACL]] parecen competir y son complementarios. OWL **infiere** bajo *mundo abierto* (lo no declarado es desconocido, no falso); SHACL **valida** bajo mundo cerrado (esto tiene que estar). Si querés que falte un dato y te avisen, SHACL. Si querés deducir hechos nuevos, OWL.

## 📝 Notas

### Fundamentos conceptuales

- [[ontología]] — qué es, qué no es, y cuándo **no** construir una. La definición de Gruber desarmada.
- [[espectro semántico]] — el continuo de expresividad que ordena todo el campo, con el mismo concepto modelado en los seis escalones.
- [[Description Logic]] — el fundamento formal detrás de OWL: TBox/ABox, los constructores, por qué OWL 2 DL es `SROIQ(D)`, y las dos suposiciones (mundo abierto, no-UNA) que rompen la intuición SQL.

### El stack de estándares

- [[RDF]] — el modelo de datos base: tripletas, IRIs, literales, blank nodes, named graphs y serializaciones.
- [[RDFS]] — la capa de esquema mínima: clases, jerarquía, `domain`/`range`. El primer escalón con inferencia.
- [[OWL]] — el lenguaje de ontologías: axiomas lógicos, clases definidas, razonamiento automático, perfiles EL/QL/RL/DL.
- [[SKOS]] — el escalón barato: taxonomías, thesauri, multilingüe. **Lo que la mayoría de los proyectos necesita y casi ninguno elige.**
- [[SPARQL]] — el lenguaje de consulta: pattern matching sobre grafos, property paths, tests ejecutables.
- [[SHACL]] — validación estructural de grafos: shapes, restricciones, informe de validación. La contraparte de mundo cerrado de OWL.

### Práctica y herramientas

- [[competency questions]] — el instrumento que ancla todo proyecto: requisitos, alcance y tests de aceptación en un solo artefacto.
- [[Ontology Design Patterns (ODP)]] — n-ario, parte-todo, rol, tiempo, value partition, procedencia y listas, con código. Más antipatrones, fundacionales y catálogos de reuso.
- [[clases insatisfacibles]] — el síntoma diagnóstico central, con tabla síntoma → causa y procedimiento de debugging.
- [[Razonadores OWL]] — ELK, HermiT, Openllet: cuál usar, perfiles y qué determina la performance real.
- [[Herramental de ontologías]] — **el mapa actualizado a 2026**: ROBOT, ODK, CI, SHACL en la práctica, triple stores, poblado de datos, y qué quedó obsoleto.
- [[IRIs y versionado]] — la decisión más cara de revertir: hash vs slash, opacos vs descriptivos, SemVer adaptado a inferencias, deprecación y migración.

### El cruce con AI

- [[Knowledge graph]] — el pariente industrial dominante; **RDF vs property graphs**, la división real de la industria.
- [[Ontología y LLMs]] — grounding, extracción asistida, y por qué "el LLM ya sabe el dominio" es un error de categoría.
- [[Graph RAG]] — recuperación sobre grafos: cuándo rinde, cuándo es sobre-ingeniería.

## 🔗 Conexiones con otros dominios

- **[[_Ontology Engineering|Ontology Engineering]]** (Libros) — la metodología de ingeniería que usa este stack: cómo se construye, evalúa y mantiene una ontología como artefacto de software.
- **[[_AI|AI]]** — el cruce vivo: [[Graph RAG]] y los [[Knowledge graph|knowledge graphs]] como fuente de grounding estructurado para sistemas LLM. Ver también [[Grounding]] y [[Hallucinations]].
- **[[_RAG|RAG]]** — [[Graph RAG]] vive ahí; se combina con [[Hybrid Search]] y [[Reranking]].

## 🗺️ Rutas de lectura

- **Entender el campo** → [[ontología]] → [[espectro semántico]] → [[Knowledge graph]]
- **Voy a modelar** → [[competency questions]] → [[OWL]] → [[Ontology Design Patterns (ODP)]] → [[clases insatisfacibles]]
- **Voy a operar** → [[Herramental de ontologías]] → [[Razonadores OWL]] → [[IRIs y versionado]] → [[SHACL]]
- **Me interesa el cruce con LLMs** → [[Knowledge graph]] → [[Ontología y LLMs]] → [[Graph RAG]]
- **¿Me alcanza con algo más barato?** → [[espectro semántico]] → [[SKOS]]

## 🌱 Conceptos para escribir

- [[Protégé]] — el editor estándar; cubierto parcialmente en [[Herramental de ontologías]]
- [[Ontologías fundacionales]] — BFO, DOLCE, SUMO; cubierto parcialmente en [[Ontology Design Patterns (ODP)]]
- Metodologías con nombre — METHONTOLOGY, NeOn, SAMOD, LOT; resumidas en [[02 - Ontology Development Methodology]]

## 🔍 Todas las notas (auto)

```dataview
TABLE file.mtime AS "Modificada"
FROM "Semantic Web"
WHERE !contains(file.tags, "type/moc")
SORT file.name ASC
```
