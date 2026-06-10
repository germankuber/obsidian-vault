---
title: AutoResearch
source: https://substack.com/home/post/p-198249792
author: Jam with AI
created: 2026-06-10
tags:
  - ai/mlops/automlops
aliases:
  - AutoResearch
  - autoresearch
  - Karpathy AutoResearch
  - The Ratchet
---

# AutoResearch

> [!note] Definición
> El setup de Karpathy donde un **agente de coding** (Claude Code, Codex o similar) lee un archivo de direcciones de investigación, edita el código de entrenamiento, corre un experimento corto, puntúa el resultado contra una **métrica congelada** y **mantiene o revierte** el cambio. Corre desatendido, de noche, sin un humano en el loop.

## El contrato — 3 archivos

- **`prepare.py`** — define la **métrica de validación** y los datos held-out. Es el **evaluador**. Ni el humano ni el agente lo editan durante un run.
- **`train.py`** — modelo, optimizer y training loop. Es el **sandbox** donde el agente puede hacer cambios.
- **`program.md`** — las direcciones de investigación, en inglés plano.

El agente lee las direcciones, edita el sandbox, entrena por una ventana fija y puntúa contra la métrica congelada. Si el score mejora, se commitea; si no, se revierte.

## El ratchet (trinquete)

> [!tip] El codebase solo avanza
> Karpathy lo llamó el **ratchet**: solo se conservan los cambios que mejoran el score. *"The clever part is not that the AI suddenly becomes a perfect researcher. The clever part is the contract."*
>
> **Un** archivo editable · **un** evaluador congelado · **una** métrica escalar · **una** regla: quedate con el cambio solo si mejora el score. Ese contrato es lo que hace posible la experimentación desatendida.

## Por qué cambia la forma de experimentar

En vez de un humano probando una idea, esperando que termine el run, chequeando y probando la siguiente a mano, el agente prueba muchos cambios chicos por su cuenta, de noche. Es el germen de [[AutoMLOps]] (su versión de producción).

## Resultados reportados

- **Red Hat**: **198 experimentos desatendidos** en OpenShift AI con **+2.3% de mejora en validation-loss**.
- **Shopify**: exploró una dirección similar, aunque algunas ganancias fueron descritas explícitamente como **"likely overfit"** — exactamente el riesgo que AutoMLOps busca controlar.

## References

- Fuente: [Harness Engineering: Evolution of MLOps to AutoMLOps](https://substack.com/home/post/p-198249792) — Jam with AI, 2026-05-21
- Repo: [karpathy/autoresearch](https://github.com/karpathy/autoresearch)

## Related

- [[AutoMLOps]]
- [[Three-Tier Evaluation Pipeline]]
- [[Offline vs Business Metrics]]
- [[Generator-Evaluator Pattern]]
- [[Sandboxing]]
