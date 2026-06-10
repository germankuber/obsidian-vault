---
title: Cursor Pagination
source: https://designgurus.substack.com/p/api-pagination-guide-cursor-vs-offset
author: Arslan Ahmad (Design Gurus)
created: 2026-06-08
tags:
  - system-design/api
aliases:
  - Cursor Pagination
  - The Bookmark Approach
---

# Cursor Pagination

> [!note] Definición
> Reemplaza el offset numérico por un **cursor opaco** que apunta a un ítem específico del result set. En vez de "saltá 40 filas y dame 20", el cliente dice **"dame 20 filas empezando después de este ítem"**. Es el "bookmark approach".

## Cómo funciona

- Cliente: `GET /api/posts?limit=20&cursor=eyJpZCI6MTIzNDV9`.
- El server decodifica el cursor (típicamente una referencia Base64 al sort key del último ítem) y construye una query que **filtra en vez de saltar**:

```sql
SELECT * FROM posts
WHERE id < 12345
ORDER BY id DESC
LIMIT 20
```

- La respuesta incluye tokens de cursor para la página siguiente/anterior:

```json
{
  "data": [...],
  "pagination": {
    "next_cursor": "eyJpZCI6MTIzMjV9",
    "has_more": true
  }
}
```

- El cliente no sabe ni le importa el número de página: solo pasa el cursor de vuelta en el próximo request.

## Por qué escala

> [!tip] Costo constante, no lineal
> La diferencia clave está en el `WHERE`. En vez de escanear y descartar filas ([[Offset Pagination|OFFSET]]), la DB usa un **índice para saltar directo** a la posición correcta. `WHERE id < 12345 ORDER BY id DESC LIMIT 20` usa un index scan igual de rápido en la primera página que en la diez-milésima.

Benchmark en la misma tabla de 10M filas: Page 1: **8ms** · Page 50: **9ms** · Page 1.000: **9ms** · Page 10.000: **9ms**. **Constante** — cada página hace el mismo trabajo: un index seek + leer 20 filas. (Comparar con offset, que llega a 8.200ms en la página 10.000.)

## Ventaja de consistencia

- Como el cursor apunta a un **ítem específico** (no a una posición), inserts/deletes no causan duplicados ni ítems salteados. Si se crea un post nuevo mientras el usuario scrollea, puede aparecer en una página futura pero **nunca corre la página actual**.

## Qué resignás

> [!warning]
> - **No random access**: el usuario no puede saltar a "página 50", solo next/previous. OK para infinite scroll, no para navegación numerada.
> - **No total count**: no podés decir eficientemente "página 3 de 771" — computar el total en tablas grandes es caro (un `COUNT(*)` aparte que puede ser lento).
> - **Más complejo de implementar**: encodear/decodear cursors, manejar edge cases (¿cursor a un ítem borrado?), y mantenerlo opaco para que el cliente no lo manipule.

## Cuándo usarlo

> [!tip]
> Cualquier endpoint con datasets grandes, UIs de infinite-scroll (mobile/web), datos que cambian seguido (feeds, timelines, notificaciones), o donde la paginación profunda es posible.

## Cursor vs Keyset

El cursor describe la **interfaz de la API**; [[Keyset Pagination]] describe la **técnica SQL** que lo hace funcionar (y lo extiende a columnas de orden no únicas con cursores compuestos). A menudo se usan como sinónimos.

## References

- Fuente: [API Pagination Guide: Cursor vs Offset vs Keyset](https://designgurus.substack.com/p/api-pagination-guide-cursor-vs-offset) — Arslan Ahmad (Design Gurus), 2026-04-20
- Mención original: [50 System Design Patterns Every Engineer Should Know](https://designgurus.substack.com/p/50-system-design-patterns-every-engineer) — Design Gurus, 2026-05-11

## Related

- [[Pagination]]
- [[Offset Pagination]]
- [[Keyset Pagination]]
- [[API Versioning]]
