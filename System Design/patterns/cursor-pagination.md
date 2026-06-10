---
title: Cursor Pagination
source: https://designgurus.substack.com/p/50-system-design-patterns-every-engineer
author: Design Gurus
created: 2026-06-08
tags:
  - system-design/api
  - system-design/patterns
aliases:
  - Cursor Pagination
  - cursor-pagination
  - Keyset Pagination
---

# Cursor Pagination

> [!note] Definition
> Paginar resultados con un **cursor opaco** que apunta al último ítem devuelto;
> el cliente pasa ese cursor para pedir la página siguiente. Alternativa al
> clásico `OFFSET/LIMIT`.

## Cómo funciona

En vez de "saltá 10000, traé 20" (offset), el cursor codifica una posición
estable (ej. el `id` o `created_at` del último ítem): "traé 20 con id > X". La
query usa el índice directamente, sin contar y descartar filas previas.

## Cuándo usarlo

> [!tip]
> Para **listas grandes** o *feeds infinitos*, y cuando los datos cambian mientras
> se pagina. Rendimiento constante sin importar qué tan profundo vayas.

## Cuándo NO usarlo / trade-offs

> [!warning]
> - **No podés saltar a una página arbitraria** ("ir a la página 50"): el cursor
>   es secuencial. Si necesitás navegación por número de página, offset es más
>   natural (a costa de rendimiento).
> - **El cursor depende de un orden estable**: necesitás una columna ordenada y
>   única; ordenamientos complejos complican el cursor.
> - Más difícil de implementar que `OFFSET/LIMIT`.
>
> Aun así, offset-pagination se degrada mal: `OFFSET 100000` obliga a la base a
> recorrer y descartar 100k filas. Para datos grandes, cursor casi siempre gana.

## Patrones relacionados / alternativas

- [[API Versioning]] — cambiar de offset a cursor suele requerir versionar la API.

## References

- Fuente: [50 System Design Patterns Every Engineer Should Know](https://designgurus.substack.com/p/50-system-design-patterns-every-engineer) — Design Gurus, 2026-05-11
- Trade-offs y guía de uso: conocimiento general, no del artículo fuente.

## Related

- [[System Design]]
- [[API Versioning]]
