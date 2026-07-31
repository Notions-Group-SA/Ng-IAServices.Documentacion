---
doc_id: REQ-FUN-001
doc_type: requirements-functional
title: Requisitos funcionales (reconstruidos) — IAConnect
version: 1.0.0
status: draft
origin: reverse-engineered
confidence: medium
owner: pendiente-asignacion
last_review: 2026-07-15
review_cycle_days: 180
audience: [dev, qa, producto, agentes-automaticos]
classification: uso-interno
traces: []
supersedes: null
---

# Requisitos funcionales (reconstruidos) — IAConnect

> **Resumen ejecutivo.** Requisitos **inferidos** desde el comportamiento del código y los casos de uso del origen
> (`docs/02_especificacion_funcional/casos-de-uso/`). Confianza media: reflejan lo que el sistema **hace**, no
> necesariamente lo que se **pidió**. Requieren validación con negocio.

| ID | Requisito | Evidencia | CU origen |
|---|---|---|---|
| RF-AUTH-01 | Un usuario se autentica con usuario/contraseña y recibe JWT + refresh token | `AuthController.Login`, `AuthService` | — |
| RF-AUTH-02 | El access token se renueva con un refresh token válido | `AuthController.Refresh` | — |
| RF-AUTH-03 | El logout revoca el refresh token | `AuthController.Logout` | — |
| RF-AUTH-04 | Tras 5 intentos fallidos la cuenta se bloquea 15 min | `AuthService` | — |
| RF-USER-01 | Un admin administra usuarios (alta/consulta/edición/baja lógica) | `AuthController` `[Authorize(admin)]` | — |
| RF-TEN-01 | Un admin administra tenants y su configuración de IA | `TenantsController` | CU-06 |
| RF-TEN-02 | La API key del proveedor por tenant se almacena cifrada | `TenantService` (AES) | CU-06 |
| RF-CHAT-01 | Chat multi-turno por tenant con memoria de sesión | `AIController.Chat`, `ChatService` | CU-01 |
| RF-CHAT-02 | El chat admite imagen si el tenant lo permite (formato/tamaño válidos) | `ImageValidator` | CU-01 |
| RF-COMP-01 | Generación de texto (completion) por prompt | `AIController.Completion` | CU-02 |
| RF-ANLZ-01 | Análisis de texto por tipo (sentimiento/entidades/categorización) | `AIController.Analyze` | CU-03 |
| RF-SUMM-01 | Resumen de un documento/texto | `AIController.Summarize` | CU-04 |
| RF-IMPR-01 | Mejora de texto según objetivo | `AIController.Improve` | CU-05 |
| RF-KNOW-01 | Carga de documentos y fragmentación para RAG por tenant | `KnowledgeController`, `KnowledgeService` | CU-07 |
| RF-KNOW-02 | Consulta de fragmentos almacenados de un tenant | `KnowledgeController.GetChunks` | CU-07 |
| RF-RAG-01 | Recuperación de contexto relevante e inyección en el prompt | `RAGEngine`, `PromptBuilder` | CU-01/07 |
| RF-MET-01 | Registro de métricas de uso por solicitud IA | `sys_Metricas_Uso` | — |
| RF-PROV-01 | La solicitud se enruta al proveedor IA configurado por el tenant | `AIProviderFactory` | — |

> **Nota RF-RAG-01:** el diseño previsto (embeddings) difiere de la implementación (TF-IDF léxico). Ver
> [ADR-0006](../04-decisions/ADR-0006-rag-por-tenant.md) y `GAP-RAG-SEMANTIC`.
