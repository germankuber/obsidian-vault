---
title: ACL Filtering en RAG
source: https://avanichaskar.substack.com/p/system-design-deep-dive-how-to-design
author: Avani Chaskar
created: 2026-06-08
tags:
  - ai/rag
  - ai/rag/retrieval
  - system-design/security
  - type/concept
  - status/permanent
aliases:
  - ACL Filtering en RAG
  - acl-filtering
  - Single-Stage Filtering
  - Permission Filtering RAG
---

# ACL Filtering en RAG

> [!note] Definition
> Hacer cumplir los permisos (Access Control List) **dentro de la consulta a la
> base vectorial**, en una sola etapa, de modo que la recuperación solo devuelva
> chunks que el usuario tiene derecho a ver. Es la "regla de oro" de un RAG
> empresarial: un usuario nunca debe recibir una respuesta derivada de un
> documento prohibido.

## Por qué en una sola etapa

La tentación ingenua es recuperar los mejores K chunks por relevancia y *después*
filtrar los que el usuario no puede ver. Eso es **post-filtrado** y es malo: si
recuperás 100 y 99 son inaccesibles, te queda 1 resultado (o ninguno) y
desperdiciaste el trabajo. Peor: invita a fugas si el filtro se olvida en algún
camino.

El **filtrado en una sola etapa** mete el permiso como predicado de la propia
búsqueda: la base vectorial solo considera candidatos accesibles desde el
principio.

## Cómo se implementa

Cada vector lleva metadata de permisos junto al contenido:

```json
{
  "chunk_id": "c_987",
  "document_id": "doc_xyz",
  "allowed_users": ["user_12", "user_45"],
  "allowed_groups": ["engineering", "leadership"]
}
```

En la consulta se pasa la identidad y los grupos del usuario, y la base filtra por
`allowed_users`/`allowed_groups` **antes** de rankear por similitud.

> [!warning]
> La firma ACL del usuario también debe formar parte de la clave de cualquier
> [[Redis Cache]] de respuestas: si no, dos usuarios con permisos distintos
> podrían compartir una respuesta cacheada y filtrarse información.

## References

- Fuente: [System Design Deep Dive: How to Design an Enterprise RAG Assistant](https://avanichaskar.substack.com/p/system-design-deep-dive-how-to-design) — Avani Chaskar, 2026-05-17

## Related

- [[Enterprise RAG Assistant]]
- [[Hybrid Search]]
