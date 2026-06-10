---
title: Write-Behind
source: https://designgurus.substack.com/p/50-system-design-patterns-every-engineer
author: Design Gurus
created: 2026-06-08
tags:
  - system-design/caching
  - system-design/patterns
aliases:
  - Write-Behind
  - write-behind
  - Write-Back
---

# Write-Behind

> [!note] Definition
> Las escrituras van **primero a la cache** y se confirman al instante; la cache
> vuelca a la base de forma **asíncrona**, normalmente en lotes. Habilita
> escrituras muy rápidas.

## Cómo funciona

La app escribe en la cache y recibe OK enseguida. Un proceso de fondo agrupa las
escrituras pendientes y las persiste en la base por lotes, amortiguando picos y
reduciendo la cantidad de operaciones a disco. La base va "por detrás" de la
cache.

## Cuándo usarlo

> [!tip]
> Cuando el **throughput de escritura** es el cuello de botella y podés tolerar
> una pequeña ventana de pérdida de durabilidad: métricas, contadores, logs,
> telemetría — escrituras altas donde perder los últimos milisegundos no es
> catastrófico.

## Cuándo NO usarlo / trade-offs

> [!warning]
> - **Riesgo de pérdida de datos**: si la cache se cae antes de volcar, perdés
>   las escrituras pendientes. Es el trade-off central — velocidad por durabilidad.
> - **Consistencia eventual**: la base no refleja lo último de inmediato; otros
>   lectores que van directo a la base ven datos viejos.
> - **Complejidad**: manejar el batch, los reintentos del flush, el orden de
>   escritura. Más difícil de operar.
> - Para datos donde **no podés perder ni una escritura** (pagos, órdenes), no.
>   Usá [[Write-Through]].

## Patrones relacionados / alternativas

- [[Write-Through]] — el opuesto: durable pero más lento.
- [[Cache-Aside]] — no participa en la escritura.

## References

- Fuente: [50 System Design Patterns Every Engineer Should Know](https://designgurus.substack.com/p/50-system-design-patterns-every-engineer) — Design Gurus, 2026-05-11
- Trade-offs y guía de uso: conocimiento general, no del artículo fuente.

## Related

- [[_System Design|System Design]]
- [[Write-Through]]
- [[Cache-Aside]]
