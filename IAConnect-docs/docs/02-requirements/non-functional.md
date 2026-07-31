---
doc_id: REQ-NFR-001
doc_type: requirements-non-functional
title: Requisitos no funcionales (reconstruidos) — IAConnect
version: 1.0.0
status: draft
origin: reverse-engineered
confidence: medium
owner: pendiente-asignacion
last_review: 2026-07-15
review_cycle_days: 180
audience: [arquitectos, dev, qa, seguridad, agentes-automaticos]
classification: uso-interno
traces: []
supersedes: null
---

# Requisitos no funcionales (reconstruidos) — IAConnect

> **Resumen ejecutivo.** RNF **inferidos** de mecanismos presentes en el código (ISO 25010 como checklist). Donde
> no hay evidencia de un objetivo cuantificado, se marca como *gap* (intención sin métrica verificable).

| ID | Categoría (ISO 25010) | Requisito / mecanismo | Evidencia | Estado |
|---|---|---|---|---|
| RNF-SEC-01 | Seguridad | Autenticación JWT firmada; expiración corta configurable | `Program.cs`, `AuthService` | Implementado |
| RNF-SEC-02 | Seguridad | Contraseñas con BCrypt; bloqueo por fuerza bruta | `AuthService` | Implementado |
| RNF-SEC-03 | Seguridad | Aislamiento de datos por tenant | middleware/filtros + `Id_Tenant` | Implementado (lógico) |
| RNF-SEC-04 | Seguridad | Secretos fuera del repo; API key de tenant cifrada (AES) | `TenantService`, config | Parcial (fallback a texto plano — `GAP-ENC-FALLBACK`) |
| RNF-SEC-05 | Seguridad | Sin exposición de secretos en respuestas | `TenantDto` sin `AiApiKey` | Implementado |
| RNF-REL-01 | Fiabilidad | Reintentos ante fallos transitorios del proveedor | `ClaudeProvider` (retry) | Parcial (solo Claude — heterogéneo) |
| RNF-REL-02 | Fiabilidad | Health checks para readiness | `/health`, compose | Implementado |
| RNF-PERF-01 | Rendimiento | Timeout de llamada al proveedor (60 s) | `Program.cs` HttpClient Claude | Parcial |
| RNF-PERF-02 | Rendimiento | Índices en columnas de filtrado por tenant | `scripts/01_create_database.sql` | Implementado |
| RNF-MAIN-01 | Mantenibilidad | Separación en capas (Clean Architecture) | referencias de proyecto | Implementado (con desviación Infra→App) |
| RNF-OBS-01 | Observabilidad | Métricas de consumo por tenant/proveedor | `sys_Metricas_Uso` | Implementado |
| RNF-OBS-02 | Observabilidad | Correlación de requests / tracing distribuido | — | **Gap** |
| RNF-USAB-01 | Usabilidad | Documentación viva del API (Swagger) | `Program.cs` | Implementado |
| RNF-A11Y-01 | Usabilidad/A11y | Accesibilidad WCAG del widget/portal | — | **Gap** (`GAP-A11Y`) |
| RNF-PORT-01 | Portabilidad | Contenerización (Docker) | `Dockerfile`, compose | Implementado |

## Observaciones

- No se hallaron **objetivos cuantificados** (SLO de latencia/disponibilidad) en el origen → cada RNF de rendimiento/
  fiabilidad debe recibir una meta verificable en revisión humana.
- El perfil regulado (A.17) **no** está activo (`solution.regulated=false`); si el dominio del cliente lo exige,
  activar mapeo regulatorio y retención reforzada.
