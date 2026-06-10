---
title: Content Delivery Network
source: https://designgurus.substack.com/p/50-system-design-patterns-every-engineer
author: Design Gurus
created: 2026-06-08
tags:
  - system-design/infrastructure
  - system-design/patterns
  - type/pattern
  - status/permanent
aliases:
  - Content Delivery Network
  - content-delivery-network
  - CDN
---

# Content Delivery Network

> [!note] Definition
> Distribuir contenido estático a **edge servers** repartidos por el mundo; cada
> usuario se sirve desde el edge más cercano, reduciendo mucho la latencia.

## Cómo funciona

El CDN cachea copias del contenido (imágenes, JS, CSS, video, a veces respuestas
de API) en POPs geográficamente distribuidos. El usuario en Buenos Aires recibe
el asset desde un edge local en vez de cruzar el planeta hasta el origen. El
origen solo se toca en el primer miss o al vencer el TTL.

## Cuándo usarlo

> [!tip]
> Para **contenido estático** servido a una audiencia geográficamente dispersa:
> sitios web, SPAs, descargas, streaming, imágenes. Baja latencia, descarga el
> origen y absorbe picos de tráfico.

## Cuándo NO usarlo / trade-offs

> [!warning]
> - **Invalidación de caché**: actualizar contenido cacheado en cientos de edges
>   no es instantáneo; un deploy puede servir assets viejos. Se maneja con
>   *cache-busting* (hash en el nombre del archivo).
> - **Contenido dinámico/personalizado** cachea mal — un CDN no ayuda con
>   respuestas únicas por usuario (salvo edge compute).
> - **Costo** y otra dependencia de terceros.
> - Para una audiencia local y poco tráfico, puede no justificarse.

## Patrones relacionados / alternativas

- [[Reverse Proxy]] — un CDN es, en esencia, un reverse-proxy-caché distribuido
  globalmente.
- [[Cache-Aside]] — misma idea de caché, a nivel de red.

## References

- Fuente: [50 System Design Patterns Every Engineer Should Know](https://designgurus.substack.com/p/50-system-design-patterns-every-engineer) — Design Gurus, 2026-05-11
- Trade-offs y guía de uso: conocimiento general, no del artículo fuente.

## Related

- [[_System Design|System Design]]
- [[Reverse Proxy]]
- [[Cache-Aside]]
