---
title: Markdown-Aware Chunking
source: https://togoailabs.substack.com/p/the-complete-guide-to-rag-chunking
author: togoAI Labs
created: 2026-06-08
tags:
  - ai/rag/ingestion
  - system-design/patterns
aliases:
  - Markdown-Aware Chunking
  - markdown-aware-chunking
  - Markdown-Aware Splitting
---

# Markdown-Aware Chunking

> [!note] Definición
> Partir respetando la **estructura del markdown**: jerarquía de headers
> (`#`, `##`, `###`), bloques de código fenced, listas y links. Cada chunk lleva
> como **metadata** su posición en la jerarquía de headers.

## Cómo funciona

- Respeta la jerarquía de headers y corta en sus límites.
- Mantiene **bloques de código intactos** (fenced con ```), preserva listas y
  links.
- Adjunta metadata de header → permite **rastrear y reconstruir** la estructura
  del documento, y filtrar por sección en el retrieval.

**Por qué los headers importan**
- Un header es un **límite de tema explícito** puesto por el autor. Todo lo que
  está bajo `## Installation` está semánticamente agrupado; cortar a mitad de
  sección pierde ese agrupamiento.

**Librería**: `MarkdownHeaderTextSplitter` de LangChain.
- Se configura con `headers_to_split_on`: lista de tuplas `(marcador, nombre_metadata)`.
- Funciona **dentro** del framework recursivo (secciones largas se sub-parten).

```python
from langchain.text_splitter import MarkdownHeaderTextSplitter
headers_to_split_on = [
    ("#", "Header 1"),
    ("##", "Header 2"),
    ("###", "Header 3"),
]
markdown_splitter = MarkdownHeaderTextSplitter(headers_to_split_on=headers_to_split_on)
# devuelve chunks con metadata de su posición en la jerarquía de headers
chunks = markdown_splitter.split_text(markdown_text)
```

## Cuándo usarlo

> [!tip]
> - Documentación técnica: READMEs, wikis.
> - Blog posts y contenido con estructura markdown.
> - Cualquier contenido markdown bien formado — la metadata de header es oro
>   para el retrieval.

## Cuándo NO usarlo / trade-offs

> [!warning]
> - Solo sirve si el contenido **ES** markdown bien formado; texto plano o HTML
>   sucio no se beneficia.
> - Secciones muy largas igual necesitan sub-split por tamaño → se combina con
>   [[Recursive Character Splitting]].
> - Secciones muy cortas → chunks diminutos sin suficiente contexto.

## References

- Fuente: [The Complete Guide to RAG Chunking Strategies (Part 1)](https://togoailabs.substack.com/p/the-complete-guide-to-rag-chunking) — togoAI Labs, 2026-02-09

## Related

- [[Chunking Strategies]]
- [[Code-Aware Chunking]]
- [[Recursive Character Splitting]]
