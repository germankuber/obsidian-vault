---
title: Recursive Character Splitting
source: https://togoailabs.substack.com/p/the-complete-guide-to-rag-chunking
author: togoAI Labs
created: 2026-06-08
tags:
  - ai/rag/ingestion
  - system-design/patterns
  - type/pattern
  - status/permanent
aliases:
  - Recursive Character Splitting
  - recursive-character-splitting
  - RecursiveCharacterTextSplitter
---

# Recursive Character Splitting

> [!note] Definición
> Probar una **jerarquía de separadores** en orden de preferencia y aplicarla
> recursivamente a cada fragmento. Se adapta a la estructura que tenga el texto,
> sin configuración manual por documento. El **estándar de facto** en 2026.

## Cómo funciona

- Jerarquía default de separadores: `"\n\n"` (párrafos) → `"\n"` (líneas) →
  `". "` (oraciones) → `" "` (palabras) → `""` (caracteres, último recurso).
- Algoritmo:
  1. Si `texto ≤ chunk_size` → devolverlo.
  2. Para cada separador de la jerarquía: si está en el texto, partir y aplicar
     recursivamente con los **separadores restantes**.
  3. Si no hay separador → forzar corte por caracteres.

```python
def recursive_split(text, separators, chunk_size):
    if len(text) <= chunk_size:
        return [text]
    for separator in separators:
        if separator in text:
            chunks = []
            for split in text.split(separator):
                chunks.extend(recursive_split(split, separators[1:], chunk_size))
            return chunks
    return character_split(text, chunk_size)  # sin separador → corte forzado
```

En la práctica (LangChain):

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter
splitter = RecursiveCharacterTextSplitter(
    chunk_size=512, chunk_overlap=100,
    separators=["\n\n", "\n", ". ", " ", ""],
)
chunks = splitter.split_text(text)
```

**Por qué funciona**
- Documentos estructurados se parten por párrafo automáticamente.
- Texto no estructurado cae a cortes por palabra.
- Se adapta a contenido diverso **en el mismo corpus**, sin reconfigurar.

## Números

- Complejidad mayor que las estrategias fijas, pero rápido para producción.
- Retrieval (corpus mixto): **74-78% MRR@10**.
- Velocidad: ~**2000 chunks/seg** (M2).
- Buena eficiencia de almacenamiento (sin procesamiento redundante).

**Librería**: `LangChain.text_splitter.RecursiveCharacterTextSplitter` (el
estándar de facto 2026 para uso general).

## Cuándo usarlo

> [!tip]
> - Documentos de **formato mixto** (docs con código + tablas + prosa).
> - Tipos de documento diversos en el mismo corpus.
> - **El default cuando no hay un requisito de dominio específico.** Buen MVP:
>   512 tokens + 20% overlap.

## Cuándo NO usarlo / trade-offs

> [!warning]
> - Requisitos estructurales muy específicos (ej. legal que necesita chunks a
>   nivel de página).
> - Cuando hay una estrategia de dominio mejor: para código, [[Code-Aware Chunking]]
>   (AST/separadores de lenguaje); para markdown, [[Markdown-Aware Chunking]].
> - Es buen **generalista, no especialista**: no gana en ningún tipo puntual.

## References

- Fuente: [The Complete Guide to RAG Chunking Strategies (Part 1)](https://togoailabs.substack.com/p/the-complete-guide-to-rag-chunking) — togoAI Labs, 2026-02-09

## Related

- [[Chunking Strategies]]
- [[Markdown-Aware Chunking]]
- [[Code-Aware Chunking]]
