---
doc_id: ADR-0006
doc_type: adr
title: "ADR-0006: Base de conocimiento por tenant (RAG)"
status: draft
origin: reverse-engineered
confidence: high
date: 2026-07-15
deciders: [pendiente-asignacion]
owner: pendiente-asignacion
last_review: 2026-07-15
audience: [arquitectos, dev, agentes-automaticos]
classification: uso-interno
traces: [ADR-0003]
supersedes: null
---

# ADR-0006: Base de conocimiento por tenant (RAG)

> Decisión reconstruida. Confianza: alta. **Contiene una divergencia relevante código↔diseño** que se reporta
> explícitamente (no se resuelve): ver más abajo.

## Contexto

Cada tenant necesita respuestas fundamentadas en su propia documentación, sin re-entrenar modelos.

## Decisión

Implementar RAG por tenant: carga de documentos (`KnowledgeService.UploadDocumentAsync`), extracción de texto
(PDF con **PdfPig**; también txt/md/html/csv), **fragmentación por ventana deslizante 400/50**, almacenamiento en
`sys_Fragmentos_Conocimiento` por tenant, y recuperación de los fragmentos más relevantes (top-K=5) que
`PromptBuilder` inyecta en el *system prompt* antes de llamar al proveedor.

## Alternativas

- **Búsqueda vectorial con embeddings + similitud coseno:** era el diseño previsto (ver «Divergencia»).
- **Sin RAG (solo system prompt):** insuficiente para conocimiento propio.

## Consecuencias

- (+) Respuestas contextualizadas por tenant sin fine-tuning.
- (–) **Divergencia diseño↔código (`GAP-RAG-SEMANTIC`):** el esquema define `Vector_Embedding varbinary(MAX)` y el
  documento de origen `docs/05_arquitectura_tecnica/rag-spec_v1.0.md` describe **embeddings + similitud coseno
  (threshold 0.75)**; el código real (`RAGEngine.cs`) implementa **recuperación léxica TF-IDF en memoria** con
  *fallback* por substring, y `KnowledgeService` guarda `Vector_Embedding = null` (el helper `SerializeEmbedding`
  es código muerto). **Gana el código:** hoy el RAG **no es semántico**. Decisión de converger (implementar
  embeddings vs. actualizar el diseño) queda para revisión humana.
- (–) La recuperación léxica puede perder relevancia semántica frente a sinónimos/paráfrasis.
