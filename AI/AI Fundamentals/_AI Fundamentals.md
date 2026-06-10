---
title: AI Fundamentals — Mapa del tema
created: 2026-06-10
tags:
  - ai/fundamentals
  - type/moc
  - status/permanent
aliases:
  - AI Fundamentals
  - AI Fundamentals MOC
  - Fundamentos de IA
updated: 2026-06-10
---

# AI Fundamentals — Mapa del tema

> [!note] Cómo usar esta nota
> Índice (MOC) de `AI/fundamentals`: conceptos **transversales** de IA/LLM que no pertenecen a un subdominio específico (no son solo de RAG, agents o MLOps) sino que los cruzan a todos. Abrí esta nota, no la carpeta.

## 🧠 Entrenamiento y alineación

- [[RLHF]] — alinear un LLM con preferencias humanas vía RL (SFT → reward model → PPO). Incluye RLAIF, DPO, Constitutional AI.

## ✅ Calidad, evaluación y confiabilidad

- [[Hallucinations]] — salida plausible pero incorrecta/no respaldada; el problema es la confianza sin incertidumbre.
- [[Grounding]] — atar la respuesta a evidencia verificable; la principal defensa contra las alucinaciones.
- [[Evals]] — tests que miden si el sistema se comporta como se espera; reducen ignorancia, no garantizan perfección.

## 🌱 Por escribir (semillas del grafo)

Conceptos base que aún no tienen nota propia — candidatos a promover cuando un artículo aporte material (varios vienen del pendiente "25 AI Concepts"):

- [[Tokens]] · [[Attention]] · [[Transformers]] · [[Embeddings]] · [[Context Window]]
- [[Fine-tuning]] · [[LoRA]] · [[Quantization]] · [[Distillation]] · [[Inference]]
- [[Prompting]] · [[Context Engineering]]

## 🔍 Todas las notas de esta carpeta (auto)

```dataview
LIST
FROM "AI/AI Fundamentals"
WHERE file.name != this.file.name
SORT file.name ASC
```
