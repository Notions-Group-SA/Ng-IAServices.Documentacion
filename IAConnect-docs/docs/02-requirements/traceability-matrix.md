---
doc_id: REQ-TRC-001
doc_type: traceability-matrix
title: Matriz de trazabilidad — IAConnect
version: 1.0.0
status: draft
origin: reverse-engineered
confidence: medium
owner: pendiente-asignacion
last_review: 2026-07-15
review_cycle_days: 180
audience: [qa, dev, auditoria, agentes-automaticos]
classification: uso-interno
traces: []
supersedes: null
---

# Matriz de trazabilidad — IAConnect

> **Resumen ejecutivo.** Conecta requisito → decisión (ADR) → código → prueba. Reconstruida; los huecos de prueba
> se marcan como *gap* (ver [pieces/IAConnect.Tests](../pieces/IAConnect.Tests/README.md) y §15
> [test-data-matrix](../03-data/test-data-matrix.md)).

| Requisito | ADR | Código (fuente) | Prueba | Caso §15 |
|---|---|---|---|---|
| RF-AUTH-01/02/03 | ADR-0005 | `AuthService`, `AuthController` | `AuthControllerIntegrationTests`, `AuthServiceTests` | TC-AUTH-* |
| RF-AUTH-04 | ADR-0005 | `AuthService` (bloqueo) | `AuthServiceTests` | TC-AUTH-lock |
| RF-USER-01 | ADR-0005 | `AuthController` CRUD | *(parcial)* | — |
| RF-TEN-01 | ADR-0003 | `TenantsController`, `TenantService` | `TenantsControllerIntegrationTests`, `TenantServiceTests` | TC-TEN-* |
| RF-TEN-02 | ADR-0003/0004 | `TenantService` (AES) | *(gap)* | — |
| RF-CHAT-01/02 | ADR-0004/0006 | `ChatService`, `ImageValidator` | `ChatServiceTests`, `ImageValidatorTests` | TC-MSG-* |
| RF-COMP-01 | ADR-0004 | `CompletionService` | `CompletionServiceTests` + integ. auth | — |
| RF-ANLZ-01 | ADR-0004 | `AnalyzeService` | `AnalyzeServiceTests` | — |
| RF-SUMM-01 | ADR-0004 | `SummarizeService` | `SummarizeServiceTests` | — |
| RF-IMPR-01 | ADR-0004 | `ImproveService` | `ImproveServiceTests` | — |
| RF-KNOW-01/02 | ADR-0006 | `KnowledgeService/Controller` | **gap (sin test)** | TC-FRG-* |
| RF-RAG-01 | ADR-0006 | `RAGEngine`, `PromptBuilder` | `RAGEngineTests` (solo fallback) | — |
| RF-PROV-01 | ADR-0004 | `AIProviderFactory` | `AIProviderFactoryTests` | TC-TEN-provider |
| RF-MET-01 | — | `sys_Metricas_Uso` (persistencia) | **gap** | TC-MET-* |
| RNF-SEC-03 | ADR-0003 | `TenantAccessFilter`, `TenantResolverMiddleware` | `MultiTenantIsolationTests`, `TenantResolverMiddlewareTests` | — |

## Gaps de trazabilidad

- **Sin cobertura de prueba:** `KnowledgeService`/`KnowledgeController`, cifrado de API key, persistencia de métricas,
  providers concretos (solo se prueba la factory), y la mayoría de endpoints IA en integración (solo `/completion`).
- `GlobalExceptionMiddleware` verificado solo para `InvalidCredentials→401`.
- La columna «Caso §15» referencia la matriz derivada del modelo de datos ([test-data-matrix](../03-data/test-data-matrix.md)).
