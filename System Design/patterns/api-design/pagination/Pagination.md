---
title: Pagination
source: https://designgurus.substack.com/p/api-pagination-guide-cursor-vs-offset
author: Arslan Ahmad (Design Gurus)
created: 2026-06-10
tags:
  - system-design/api
aliases:
  - Pagination
  - API Pagination
---

# Pagination

> [!note] Definición
> Cómo una API devuelve listas grandes en pedazos. El patrón que elegís decide si tu API sigue rápida a 100 millones de filas o colapsa. Tres patrones: [[Offset Pagination]], [[Cursor Pagination]], [[Keyset Pagination]].

## Por qué importa a escala

- Todo anda bien hasta que el dataset crece. En dev, 500 filas: `LIMIT 20 OFFSET 0` responde en 2ms. Seis meses después, 50M filas: un usuario scrollea a la página 5.000, la API manda `LIMIT 20 OFFSET 99980`, PostgreSQL escanea 99.980 filas, las descarta, y la query tarda 8 segundos. La app hace timeout.
- *"Las primeras 50 páginas son rápidas. La página 500 es lenta. La página 5.000 es inusable."*
- Y para cuando lo notás, tus clientes ya están integrados contra tu interfaz de paginación → cambiarla duele.

## Los tres patrones

- [[Offset Pagination]] — `LIMIT/OFFSET`. Simple, random access, total count. Se degrada linealmente (escanea y descarta). Bueno para datasets chicos con números de página.
- [[Cursor Pagination]] — cursor opaco que apunta a un ítem. Rendimiento **constante** (index seek). El default para datasets grandes/cambiantes. Sin random access ni total count.
- [[Keyset Pagination]] — el technique a nivel DB que hace funcionar al cursor; extiende a compound sort keys (columnas no únicas) con `(sort_key, id)`.

## El decision framework (4 preguntas)

> [!tip]
> 1. **¿Qué tan grande es el dataset?** < 100k filas → offset OK. > 100k → cursor o keyset.
> 2. **¿La UI necesita números de página?** Sí → offset (o híbrido). Infinite scroll / "load more" → cursor o keyset.
> 3. **¿Los datos cambian entre requests?** Inserts/deletes frecuentes → offset produce duplicados y huecos; cursor/keyset lo manejan bien.
> 4. **¿Ordenás por columna no única?** Solo por ID único → cursor simple. Por timestamp/precio/etc → keyset con cursor compuesto.
>
> Para la mayoría de APIs modernas (mobile, SPAs, exports a escala), **cursor + keyset es el default más fuerte**. Empezá con offset solo si sabés que el dataset queda chico y la UI requiere números de página.

## Híbridos del mundo real

- **Offset para páginas tempranas, cursor para profundas**: offset en páginas 1–100 (perf aceptable), cambio automático a cursor cuando el offset cruza un umbral. El cliente no necesita saber del switch si la API lo abstrae.
- **Cursor con total estimado**: `pg_class.reltuples` da un conteo aproximado rápido sin `COUNT(*)` completo. Muestra "aproximadamente 15.000 resultados" en tiempo constante. Actualizado por `ANALYZE`, exacto dentro de unos pocos %.

```sql
SELECT reltuples::bigint AS estimate
FROM pg_class
WHERE relname = 'posts';
```

- **Materialized ID lists para snapshot consistency**: para cuando el cliente debe ver un result set estable entre páginas (ej. paginar un export), materializás la lista completa de IDs en una tabla temporal/cache en el primer request; las páginas siguientes leen del snapshot → cero duplicados/huecos. Trade-off: costo de storage y staleness (no refleja cambios posteriores al primer request).

## En system design interviews

La paginación aparece en casi toda entrevista que liste datos ("design a news feed", "e-commerce search", "notification system"). El guión fuerte:

> *"Para el feed usaría cursor-based pagination con keyset queries. Cada respuesta incluye un next_cursor que codifica el timestamp e ID del último ítem. La query usa un índice compuesto en (created_at DESC, id DESC) así cada página cuesta lo mismo sin importar la profundidad. No usaría offset porque los datos del feed cambian constantemente: inserts causarían duplicados y skips al scrollear."*

Anticipá el follow-up "¿y si quieren mostrar números de página?": *"Offset con un máximo de páginas — los usuarios rara vez pasan de la página 10 en una búsqueda, así que offset es aceptable para páginas 1–200; más allá, devolver un error 'too deep' y sugerir refinar la búsqueda."*

## Errores comunes

> [!warning]
> 1. **Offset por defecto sin pensar en escala** — debe ser una elección consciente. Preguntá siempre: ¿qué pasa en la página 10.000?
> 2. **Exponer IDs crudos de la DB como cursors** — el cliente empieza a construirlos a mano y se acopla a tu schema. Base64-encodeá siempre.
> 3. **Ordenar por columna no única sin tiebreaker** — filas con igual timestamp dan orden inconsistente entre páginas. Incluí siempre un tiebreaker único.
> 4. **Faltar el índice compuesto** — keyset sin índice que matchee el `ORDER BY` es más lento que offset (cae a seq scan).
> 5. **Correr `COUNT(*)` en cada request** — contar 50M filas tarda segundos. Usá conteo estimado o cacheado.
> 6. **No enforzar un tamaño máximo de página** — si el cliente puede pedir `limit=10000`, un solo call devuelve un payload enorme; tope server-side (típico 100). También frena scraping.
> 7. **Olvidar la paginación hacia atrás** — la mayoría de cursores solo soportan "next". Para "previous", invertí el operador de comparación y el orden, y revertí los resultados antes de devolver.
> 8. **No manejar cursors a ítems borrados** — validar siempre los valores decodificados server-side y devolver un fallback sensato (primera página) si el cursor es inválido.
> 9. **Ignorar rate limiting en deep pagination** — sin límite, se puede scrapear todo el dataset. Combiná paginación con [[Rate Limiting]].

## References

- Fuente: [API Pagination Guide: Cursor vs Offset vs Keyset](https://designgurus.substack.com/p/api-pagination-guide-cursor-vs-offset) — Arslan Ahmad (Design Gurus), 2026-04-20

## Related

- [[Offset Pagination]]
- [[Cursor Pagination]]
- [[Keyset Pagination]]
- [[Rate Limiting]]
- [[API Versioning]]
