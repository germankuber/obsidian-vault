---
title: Code-Aware Chunking
source: https://togoailabs.substack.com/p/the-complete-guide-to-rag-chunking
author: togoAI Labs
created: 2026-06-08
tags:
  - ai/rag/ingestion
  - system-design/patterns
  - type/pattern
  - status/permanent
aliases:
  - Code-Aware Chunking
  - code-aware-chunking
  - Code-Aware Splitting
updated: 2026-06-11
---

# Code-Aware Chunking

> [!note] Definición
> Partir código respetando sus **límites sintácticos**: fronteras de función/ clase, bloques completos, indentación, e imports/docstrings relevantes. Separadores específicos por lenguaje.

## Cómo funciona

- Corta en fronteras de **función/clase**, no en posiciones arbitrarias.
- Preserva bloques de código completos y el **contexto de indentación**.
- Incluye imports y docstrings relevantes junto al código.
- Usa jerarquías de separadores **por lenguaje**.

**Por qué funciona**
- Una función/clase es la unidad semántica del código, igual que el párrafo lo es de la prosa. Traer media función es inútil para el retrieval.

**Librería**: `RecursiveCharacterTextSplitter.from_language` de LangChain.
- Separadores de ejemplo (Python): `"\nclass "`, `"\ndef "`, `"\n\n"`, `"\n"`.
- Configuración automática por lenguaje (Python, JavaScript, etc.).

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter

python_splitter = RecursiveCharacterTextSplitter.from_language(
    language="python",
    chunk_size=512,
    chunk_overlap=50,
)  # usa separadores como: "\nclass ", "\ndef ", "\n\n", "\n"
```

## Cuándo usarlo

> [!tip]
> - Indexado de **repositorios de código** y sistemas de code search.
> - Documentación para desarrolladores y bases de conocimiento con código embebido.

## Cuándo NO usarlo / trade-offs

> [!warning]
> - Es **por lenguaje**: necesitás los separadores correctos; un lenguaje no soportado cae al split genérico.
> - Funciones gigantes igual exceden el límite y hay que sub-partirlas, perdiendo parte del beneficio.
> - Para grafos de dependencias entre archivos esto solo no alcanza → ahí entra un enfoque tipo [[GraphRAG]].

## References

- Fuente: [The Complete Guide to RAG Chunking Strategies (Part 1)](https://togoailabs.substack.com/p/the-complete-guide-to-rag-chunking) — togoAI Labs, 2026-02-09

## Related

- [[Chunking Strategies]]
- [[Recursive Character Splitting]]
- [[Markdown-Aware Chunking]]
