---
title: Home
created: 2026-06-10
tags:
  - type/moc
  - status/permanent
aliases:
  - Home
  - Inicio
  - Index
---

# 🏠 Home — Índice del vault

> [!note] Punto de entrada
> Esta es la nota raíz del vault. Desde acá llegás a los dos dominios y a los dashboards vivos (se actualizan solos con Dataview). Empezá por el dominio que te interese y bajá por su MOC.

## 🗂️ Dominios

- [[_System Design|System Design]] — 50 patrones de diseño de sistemas (almacenamiento, caching, comunicación, confiabilidad, escalado, consistencia, observabilidad…).
- [[_AI Agents|AI Agents]] · [[_RAG|RAG]] · [[_MLOps|MLOps]] · [[_GNN|GNN]] · [[_AI Fundamentals|AI Fundamentals]] — los subdominios de **IA**: harnesses/agentes, retrieval-augmented generation, MLOps/AutoMLOps, graph neural networks e interpretabilidad, y los fundamentos transversales.

## 📚 Meta

- [[_Imported Sources|Imported Sources]] — registro de todos los artículos importados al vault (antes de importar una URL, se valida acá).

## 📊 Dashboard

### Notas por dominio

```dataview
TABLE length(rows) AS "Notas"
FROM "System Design" OR "AI"
WHERE !contains(file.tags, "type/moc")
GROUP BY file.folder AS "Carpeta"
SORT length(rows) DESC
```

### 🏷️ Notas por tipo

```dataview
TABLE length(rows) AS "Notas"
FROM "System Design" OR "AI"
FLATTEN file.tags AS tag
WHERE startswith(tag, "type/") AND tag != "type/moc"
GROUP BY tag AS "Tipo"
SORT length(rows) DESC
```

### 🚧 Notas a medias (`status/stub`)

Notas que existen pero están incompletas o aún sin fuente externa — lo que falta terminar o validar.

```dataview
TABLE file.folder AS "Carpeta"
FROM #status/stub
SORT file.name ASC
```

### 🕒 Últimas notas editadas

```dataview
TABLE updated AS "Editada", file.folder AS "Carpeta"
FROM "System Design" OR "AI"
WHERE !contains(file.tags, "type/moc") AND updated
SORT updated DESC
LIMIT 10
```

### 🆕 Últimas notas creadas

```dataview
TABLE created AS "Creada", file.folder AS "Carpeta"
FROM "System Design" OR "AI"
WHERE !contains(file.tags, "type/moc")
SORT created DESC
LIMIT 10
```

### 🌱 Semillas del grafo (notas enlazadas que aún no existen)

Conceptos referenciados con `[[wikilink]]` desde alguna nota pero que todavía no tienen archivo propio — la lista de "qué escribir después".

```dataview
TABLE length(rows) AS "Veces enlazada"
FROM "System Design" OR "AI"
FLATTEN file.outlinks AS link
WHERE !link.path
GROUP BY link AS "Semilla"
SORT length(rows) DESC
```

## 🔢 Resumen

```dataview
TABLE length(rows) AS "Total"
FROM "System Design" OR "AI"
WHERE !contains(file.tags, "type/moc")
GROUP BY "Notas de conocimiento" AS ""
```
