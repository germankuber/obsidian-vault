---
title: Keyset Pagination
source: https://designgurus.substack.com/p/api-pagination-guide-cursor-vs-offset
author: Arslan Ahmad (Design Gurus)
created: 2026-06-10
tags:
  - system-design/api
aliases:
  - Keyset Pagination
  - Seek Pagination
  - Seek Method
---

# Keyset Pagination

> [!note] Definición
> El technique a **nivel de base de datos** (también llamado *seek pagination*) que hace funcionar a la [[Cursor Pagination|cursor pagination]]. Distinción sutil: **"cursor" describe la interfaz de la API; "keyset" describe la técnica SQL.** A menudo se usan como sinónimos.

## Cómo difiere del cursor simple

- El cursor simple funciona cuando paginás por **una sola columna única** (como un `id` autoincremental).
- Keyset lo extiende a **compound sort keys** — esencial cuando ordenás por una columna **no única** como `created_at`.
- **El problema de ordenar solo por `created_at`**: varias filas pueden tener el mismo timestamp. Si usás `WHERE created_at < '2026-01-15 10:00:00'` como condición del cursor, podés **saltear o duplicar** filas que comparten ese timestamp.

## La solución: composite cursor + row value comparison

Se usa un **cursor compuesto** que incluye la columna de orden **y un tiebreaker** (normalmente la primary key):

```sql
SELECT * FROM posts
WHERE (created_at, id) < ('2026-01-15 10:00:00', 12345)
ORDER BY created_at DESC, id DESC
LIMIT 20
```

La comparación compuesta `(created_at, id) < (value, value)` se llama **row value comparison**. PostgreSQL la maneja eficientemente con un índice compuesto en `(created_at DESC, id DESC)`.

## El requisito del índice

> [!warning] Sin índice compuesto, no hay performance
> Keyset solo rinde si las columnas de orden están **indexadas juntas**. Sin un índice compuesto que matchee tu `ORDER BY`, la DB cae a un **sequential scan** y perdés todo el beneficio.

```sql
CREATE INDEX idx_posts_pagination
ON posts (created_at DESC, id DESC);
```

Después de crearlo, verificá con `EXPLAIN ANALYZE` que la query use un **index scan**, no un sequential scan. Si ves "Seq Scan" en la salida, el índice no se está usando y hay que investigar por qué.

## Encoding del cursor

El cursor debe ser **opaco** al cliente. Base64-encodeá los valores del sort key para que el cliente no pueda construir, modificar ni asumir nada sobre el formato:

```text
Cursor value: {"created_at": "2026-01-15T10:00:00Z", "id": 12345}
Encoded: eyJjcmVhdGVkX2F0IjoiMjAyNi0wMS0xNVQxMDowMDowMFoiLCJpZCI6MTIzNDV9
```

Del lado servidor: decodificar, **validar** los valores, y usarlos en el `WHERE`. Si el cursor es inválido o apunta a una fila borrada, devolver la primera página o un error apropiado.

## Cuándo es esencial

> [!tip]
> Keyset es esencial cuando ordenás por una **columna no única** (timestamps, scores, orden alfabético), el dataset tiene **millones de filas**, y los datos se insertan/actualizan activamente. Casos: feeds de social media, listados de e-commerce ordenados por precio/rating, históricos de transacciones financieras.

## References

- Fuente: [API Pagination Guide: Cursor vs Offset vs Keyset](https://designgurus.substack.com/p/api-pagination-guide-cursor-vs-offset) — Arslan Ahmad (Design Gurus), 2026-04-20

## Related

- [[Pagination]]
- [[Cursor Pagination]]
- [[Offset Pagination]]
