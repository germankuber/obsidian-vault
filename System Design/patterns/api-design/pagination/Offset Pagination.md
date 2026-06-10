---
title: Offset Pagination
source: https://designgurus.substack.com/p/api-pagination-guide-cursor-vs-offset
author: Arslan Ahmad (Design Gurus)
created: 2026-06-10
tags:
  - system-design/api
aliases:
  - Offset Pagination
  - Offset-Based Pagination
  - LIMIT OFFSET
---

# Offset Pagination

> [!note] Definición
> El patrón de paginación por defecto: el cliente pide un número de página y un tamaño, y el servidor lo traduce a SQL con `LIMIT` y `OFFSET`. Simple y familiar, pero **se rompe a escala**.

## Cómo funciona

- Cliente: `GET /api/posts?page=3&limit=20`.
- Servidor: `SELECT * FROM posts ORDER BY created_at DESC LIMIT 20 OFFSET 40`.
- La DB **saltea las primeras 40 filas** y devuelve las 20 siguientes. Page 1 = `OFFSET 0`, Page 2 = `OFFSET 20`, Page 3 = `OFFSET 40`.
- La respuesta suele incluir el total para renderizar números de página:

```json
{
  "data": [...],
  "page": 3,
  "per_page": 20,
  "total": 15420
}
```

## Por qué los ingenieros lo aman

- **Simplicidad**: implementación trivial — el server multiplica para obtener el offset. Cualquier junior lo arma en 15 minutos.
- **Random access**: el usuario puede saltar directo a la página 50 o 500. Permite navegación por números de página clickeables — algo que cursor NO puede hacer.
- **Total count**: podés mostrar "página 3 de 771" porque sabés el total de filas.

## El performance trap

> [!warning] OFFSET escanea y descarta
> Cuando escribís `OFFSET 99980`, la DB **no salta mágicamente** a la fila 99.981. Escanea desde el principio, procesa 99.980 filas, **las tira todas**, y devuelve las 20 siguientes. Cada fila antes del offset se lee y se descarta. **El costo crece linealmente con el offset.**

Benchmark en una tabla PostgreSQL de 10 millones de filas:

- Page 1 (OFFSET 0): **8ms**
- Page 50 (OFFSET 980): **45ms**
- Page 1.000 (OFFSET 19.980): **890ms**
- Page 10.000 (OFFSET 199.980): **8.200ms** ← escanea y descarta casi 200.000 filas para devolver 20.

La DB hace trabajo real por cada fila descartada: leer de disco (o buffer cache), chequear visibilidad, y evaluar el `ORDER BY`.

## El problema de consistencia

> [!warning]
> Offset se rompe cuando los datos cambian entre requests. Si el usuario está en la página 2 y se inserta una fila al tope del result set, **cada página siguiente se corre una posición**: el usuario ve un ítem duplicado (el último de la página 2 reaparece como primero de la 3) o se saltea un ítem. Para datasets estáticos (reportes, exports) no importa; para feeds en vivo (social, notificaciones, transacciones) crea una UX confusa.

## Cuándo está bien

> [!tip]
> Offset es la elección correcta cuando: dataset chico (**< 100.000 filas**), el usuario necesita navegación por número de página ("ir a página 15"), hay que mostrar el total, y los datos **no cambian** mucho entre requests. Buenos casos: dashboards de admin, resultados de búsqueda con pocos matches, reportes paginados.

## Alternativas

- [[Cursor Pagination]] — rendimiento constante para datasets grandes/cambiantes.
- [[Keyset Pagination]] — el motor a nivel DB que escala.

## References

- Fuente: [API Pagination Guide: Cursor vs Offset vs Keyset](https://designgurus.substack.com/p/api-pagination-guide-cursor-vs-offset) — Arslan Ahmad (Design Gurus), 2026-04-20

## Related

- [[Pagination]]
- [[Cursor Pagination]]
- [[Keyset Pagination]]
