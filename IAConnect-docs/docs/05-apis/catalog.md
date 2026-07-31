---
doc_id: API-CAT-001
doc_type: api-catalog
title: Catálogo de APIs — IAConnect
version: 1.0.0
status: draft
origin: reverse-engineered
confidence: high
owner: pendiente-asignacion
last_review: 2026-07-15
review_cycle_days: 90
audience: [dev, integradores, agentes-automaticos]
classification: uso-interno
traces: [ADR-0003, ADR-0004]
supersedes: null
---

# Catálogo de APIs — IAConnect

> **Resumen ejecutivo.** IAConnect expone **una** API REST (`IAConnect.API`). No hay eventos asíncronos ni otras
> APIs públicas. El contrato inferido está en [`openapi.yaml`](openapi.yaml).

## API registrada

| API | Owner | Versión | Ambientes | Estado | Contrato | Auth |
|---|---|---|---|---|---|---|
| IAConnect API | pendiente | 1.0.0 | dev `:5051/:7167`, contenedor `:8080` | activa | [openapi.yaml](openapi.yaml) | JWT Bearer |

## Convenciones observadas (§6)

| Aspecto | Observado |
|---|---|
| Base path | `/api` (`/api/auth`, `/api/ai/{tenantId}`, `/api/tenants`, `/api/tenants/{tenantId}/knowledge`) |
| Versionado | **Sin** versionado en la URL (no hay `/v1/`) → mejora sugerida |
| Formato de error | Propietario `{ error, statusCode }` — **no** RFC 9457 Problem Details |
| Seguridad | JWT Bearer; roles `admin`/`operador`; filtro por tenant en `/api/ai/{tenantId}` |
| Idempotencia | Sin `Idempotency-Key` (no hay operaciones de pago/reserva) |
| Paginación | No observada en los listados (usuarios, tenants, fragmentos) → gap para volúmenes grandes |
| Correlación | Sin `X-Correlation-Id` → gap de trazabilidad |
| Documentación viva | Swagger UI habilitado en **todos** los entornos (`/swagger`) |

## Endpoints (resumen)

Ver detalle y ejemplos en [`openapi.yaml`](openapi.yaml) y en la ia-db
[`03_api-endpoints.md`](../../../ia-db/indexes/03_api-endpoints.md).

| Grupo | Endpoints |
|---|---|
| Auth | `POST /login`, `POST /refresh`, `POST /logout`, `GET/POST /usuarios`, `GET/PUT/DELETE /usuarios/{id}` |
| AI | `POST /ai/{tenantId}/{chat,completion,analyze,summarize,improve}` |
| Tenants | `GET/POST /tenants`, `GET/PUT/DELETE /tenants/{tenantId}` |
| Knowledge | `POST/GET /tenants/{tenantId}/knowledge` |
| Health | `GET /health`, `GET /` |

## Divergencias contrato ↔ código (reportadas, §6)

- **No existe `openapi.yaml` en el origen** con versión; el de este catálogo es **inferido**. Recomendación:
  adoptar API-first y validar con Spectral + tests de contrato en CI.
- `SummarizeRequestDto` expone solo `document` (el comentario del controlador menciona «longitud máxima», que
  **no** está en el DTO).
- `AiApiKey` se recibe en `Create/UpdateTenantRequest` pero **no** se devuelve en `TenantDto` (correcto: no exponer secreto).
- Formato de error propietario en lugar de Problem Details (RFC 9457).
