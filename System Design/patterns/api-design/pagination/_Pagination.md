---
title: Pagination — Mapa del tema
created: 2026-06-10
tags:
  - system-design/api
  - moc
aliases:
  - Pagination MOC
  - Pagination Index
---

# Pagination — Mapa del tema

> [!note] Cómo usar esta nota
> Índice del subtema *pagination* dentro de [[_API Design|API Design]]. Empezá por el hub y bajá a los tres patrones. Abrí esta nota, no la carpeta.

## 🚀 Empezá por acá

- [[Pagination]] — **el hub.** Por qué importa a escala, el decision framework (4 preguntas), híbridos, errores comunes, y cómo discutirlo en system design interviews.

## Los tres patrones (de simple a escalable)

- [[Offset Pagination]] — `LIMIT/OFFSET`. Simple, random access, total count; se degrada linealmente (8ms → 8.200ms en página 10.000). Bueno para < 100k filas con números de página.
- [[Cursor Pagination]] — cursor opaco, rendimiento **constante** (9ms siempre). El default para datasets grandes/cambiantes.
- [[Keyset Pagination]] — el motor a nivel DB del cursor; compound sort keys `(created_at, id)` para columnas no únicas + índice compuesto.

## 🔗 Conexión

- Subtema de [[_API Design|API Design]] · se combina con [[Rate Limiting]] (contra scraping en deep pagination) y suele requerir [[API Versioning]] al migrar de offset a cursor.

## 🔍 Todas las notas de esta carpeta (auto)

```dataview
LIST
FROM "System Design/patterns/api-design/pagination"
WHERE file.name != this.file.name
SORT file.name ASC
```
