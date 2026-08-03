---
title: Semantic Web — MOC
created: 2026-08-03
tags:
  - semantic-web
  - type/moc
  - status/stub
aliases:
  - Semantic Web
  - Web Semántica
  - Semantic Web MOC
updated: 2026-08-03
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

- [[SPARQL]] — el lenguaje de consulta: pattern matching sobre grafos, property paths, tests ejecutables.
- [[competency questions]] — el instrumento que ancla todo proyecto de ontología: requisitos, alcance y tests de aceptación en un solo artefacto.

> [!note] Dominio en construcción. Los conceptos del stack —[[RDF]], [[RDFS]], [[OWL]], [[SKOS]], [[SHACL]], [[Knowledge graph]]— están sembrados como enlaces sin resolver desde las notas de [[_Ontology Engineering|Ontology Engineering]] y son los próximos candidatos a nota propia.

## 🔗 Conexiones con otros dominios

- **[[_Ontology Engineering|Ontology Engineering]]** (Libros) — la metodología de ingeniería que usa este stack: cómo se construye, evalúa y mantiene una ontología como artefacto de software.
- **[[_AI|AI]]** — el cruce vivo: [[Graph RAG]] y los [[Knowledge graph|knowledge graphs]] como fuente de grounding estructurado para sistemas LLM.

## 🌱 Conceptos para escribir

- [[RDF]] · [[RDFS]] — el modelo de grafo y su esquema básico
- [[OWL]] — lógicas descriptivas, perfiles EL/QL/RL, clases primitivas vs definidas
- [[SKOS]] — el escalón barato para taxonomías y thesauri
- [[SHACL]] — validación estructural, la contraparte de OWL
- [[Knowledge graph]] — el pariente industrial dominante; vale una nota que contraste ambos términos
- [[espectro semántico]] — el continuo de expresividad que ordena todo el campo
- [[Description Logic]] — el fundamento formal detrás de OWL

## 🔍 Todas las notas (auto)

```dataview
TABLE file.mtime AS "Modificada"
FROM "Semantic Web"
WHERE !contains(file.tags, "type/moc")
SORT file.name ASC
```
