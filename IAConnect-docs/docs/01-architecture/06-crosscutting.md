---
doc_id: ARCH-CC-001
doc_type: architecture-crosscutting
title: Conceptos transversales — IAConnect
version: 1.0.0
status: draft
origin: reverse-engineered
confidence: high
owner: pendiente-asignacion
last_review: 2026-07-15
review_cycle_days: 180
audience: [arquitectos, dev, ops, seguridad, agentes-automaticos]
classification: uso-interno
traces: [ADR-0003, ADR-0004, ADR-0005]
supersedes: null
---

# Conceptos transversales — IAConnect

> **Resumen ejecutivo.** Seguridad (JWT + roles + aislamiento de tenant), cifrado de secretos, manejo de errores,
> logging, resiliencia hacia proveedores y configuración. Todo verificado contra el código.

## Autenticación y autorización

| Aspecto | Implementación | Fuente |
|---|---|---|
| Autenticación | JWT Bearer, firma HmacSha256; valida issuer/audience/lifetime/firma, `ClockSkew=0` | `API/Program.cs`, `Application/Services/AuthService.cs` |
| Hash de contraseña | **BCrypt** | `AuthService.cs` |
| Bloqueo | **5 intentos fallidos → bloqueo 15 min** (`AccountLockedException`→423) | `AuthService.cs` |
| Refresh tokens | 64 bytes aleatorios, rotación, revocables (`sys_Refresh_Tokens`) | `AuthService.cs` |
| Autorización por rol | `[Authorize(Roles="admin")]` (tenants, knowledge, CRUD usuarios) | Controllers |
| Autorización por tenant | `TenantAccessFilter`: admin → cualquier tenant; operador → solo el propio (`id_tenant` claim) | `API/Middleware/TenantAccessFilter.cs` |

## Multi-tenancy

- `tenantId` viaja en la ruta (`/api/ai/{tenantId}/…`, `/api/tenants/{tenantId}/knowledge`).
- `TenantResolverMiddleware` carga el tenant (`ILutTenantsDataManager.GetOneAsync`) y **corta con 404 si no existe
  o está inactivo**; guarda el tenant en `HttpContext.Items["Tenant"]`.
- Aislamiento de datos por `Id_Tenant` en todas las tablas de negocio.
- Configuración por tenant (`lut_Tenants`): proveedor IA, modelo, system prompt, temperatura, límites de tokens e
  imagen, expiración de tokens, y **API key del proveedor** (cifrada, ver abajo).

## Cifrado de secretos

- La **API key del proveedor por tenant** (`lut_Tenants.ApiKey_IA`) se cifra con **AES-256-CBC/PKCS7**; la clave se
  toma de la variable de entorno **`IACONNECT_ENCRYPTION_KEY`** y el IV va en los primeros 16 bytes del cifrado.
  Fuente: `Application/Services/TenantService.cs` + `Infrastructure`.
- **⚠ Riesgo (reportado):** si `IACONNECT_ENCRYPTION_KEY` no está presente, el código **cae a texto plano** →
  la API key quedaría sin cifrar en la BD. Requiere decisión/hardening humano (gap `GAP-ENC-FALLBACK`).
- **⚠ Config muerta:** `Encryption:AesKey` de `appsettings.json` **no se usa** (el código usa la env var).

## Manejo de errores (mapeo excepción → HTTP)

`GlobalExceptionMiddleware` centraliza el mapeo (fuente: `API/Middleware/GlobalExceptionMiddleware.cs`):

| Excepción de dominio | HTTP |
|---|---|
| `TenantNotFoundException` | 404 |
| `InvalidCredentialsException` | 401 |
| `AccountLockedException` | **423 Locked** |
| `ImageNotAllowedException` | 400 |
| `ProviderUnavailableException` | **502 Bad Gateway** |
| `ArgumentException` | 400 |
| (otras) | 500 |

Formato de respuesta de error: `{ error, statusCode }`. Los 5xx se loguean como `Error`; el resto como `Warning`.

> **Nota (§6).** No hay `application/problem+json` (RFC 9457); el formato de error es propietario. Recomendación de
> alinear con Problem Details registrada como mejora, no como defecto.

## Logging

- `ILogger` estándar de ASP.NET Core; niveles en `appsettings.json` (`Default: Information`, `Microsoft.AspNetCore: Warning`).
- Sin correlación (`X-Correlation-Id`) ni tracing distribuido observado → gap de observabilidad.

## Resiliencia hacia proveedores IA

- **Claude:** `HttpClient` nombrado (timeout 60 s) + **retry propio** (3 intentos, backoff 2^n) ante 429/503/502/504.
- **Gemini / OpenAI:** vía SDK; retry desparejo (OpenAI sin retry propio). Heterogeneidad reportada como gap.
- Errores de proveedor → `ProviderUnavailableException` → 502.

## Métricas de uso

Cada invocación IA registra `sys_Metricas_Uso` (tokens prompt/respuesta, total, proveedor, modelo, `Duracion_Ms`).
Base para gobierno de costos.

## CORS

- `Cors:AllowedOrigins` (default `http://localhost:3000`), cualquier método/header sobre esos orígenes.

## Divergencias transversales relevantes (para revisión humana)

1. **RAG no semántico** (TF-IDF, no embeddings) pese al esquema `Vector_Embedding` y al `rag-spec` del origen → ver [ADR-0006](../04-decisions/ADR-0006-rag-por-tenant.md).
2. **Inversión de capa:** `IAConnect.Infrastructure.csproj` referencia a `IAConnect.Application` (Infra→App), contra la regla de dependencia de Clean Architecture.
3. **Entidades con `string` en lugar de enums** (`Tenant.ProveedorIA`, `Usuario.Rol`, `Mensaje.Rol`).
4. **`DefaultModel` de config no consumido:** el modelo efectivo sale de `lut_Tenants.Nombre_Modelo`, no de `AIProviders:*`.
