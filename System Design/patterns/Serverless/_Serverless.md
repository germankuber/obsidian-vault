---
title: Serverless — Mapa del tema
source: https://designgurus.substack.com/p/the-fundamentals-of-serverless-system
author: Arslan Ahmad (Design Gurus)
created: 2026-06-10
tags:
  - system-design/serverless
  - type/moc
  - status/permanent
aliases:
  - Serverless MOC
  - Serverless Index
updated: 2026-06-10
---

# Serverless — Mapa del tema

> [!note] Cómo usar esta nota
> Índice del subtema *serverless* dentro de [[_System Design|System Design]]: ejecutar código sin gestionar servidores físicos. Empezá por [[Serverless]] y bajá. Abrí esta nota, no la carpeta.

## 🧭 Fundamentos

- [[Serverless]] — qué es (y qué NO: el hardware sigue existiendo), el problema del capacity planning, y la división de labores provider/developer. El punto de partida.

## 📈 Mecánica de escalado

- [[Scaling to Zero]] — remover todo el cómputo cuando no hay tráfico → costo cero. La innovación central. Incluye el escalado hacia arriba vía concurrency.

## ⏱️ Performance: la latencia de arranque

- [[Cold Start]] — la latencia de crear un entorno de ejecución nuevo desde cero. El trade-off principal del modelo.
- [[Warm Start]] — reusar un contenedor congelado para ejecutar en milisegundos; + provisioned concurrency para eliminar cold starts (con costo).

## 🧱 Restricciones de diseño

- [[Stateless]] — las funciones no retienen estado local entre ejecuciones; obliga a usar una DB externa para todo dato persistente.

## ⚙️ Plataforma y operación

- [[AWS Lambda]] — el servicio serverless de AWS: cómo aloca contenedores, el billing pay-per-execution por milisegundos, y el puzzle memoria↔CPU.
- [[Infrastructure as Code]] — gestionar cientos de recursos cloud con archivos de texto versionables (AWS CloudFormation).

## 🔗 Conexión con el resto del grafo

- Subtema de [[_System Design|System Design]]. Cruza con [[Event-Driven Architecture]] (el paradigma del serverless: el código corre por triggers), [[Auto-Scaling]] / [[Horizontal Scaling]] (el scale-out por concurrency), y [[API Gateway]] (la puerta de entrada a las funciones).
- ⚠️ Sin relación con [[Lambda Architecture]] (patrón de datos batch+stream, comparte solo el nombre con [[AWS Lambda]]).

## 🌱 Por escribir (semillas del grafo)

- [[Concurrency]] — la ejecución simultánea de N entornos en paralelo; candidata a promover si un futuro artículo la profundiza.
- [[Virtual Machine]] · [[Runtime]] · [[CloudFormation]] — conceptos base mencionados, enlazables desde las notas de este cluster.

## 🔍 Todas las notas de esta carpeta (auto)

```dataview
LIST
FROM "System Design/patterns/Serverless"
WHERE file.name != this.file.name
SORT file.name ASC
```
