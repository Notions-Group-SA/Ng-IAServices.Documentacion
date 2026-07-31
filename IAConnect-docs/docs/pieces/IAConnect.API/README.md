---
doc_id: PIECE-API-001
doc_type: piece-readme
title: IAConnect.API — Web API REST (servicio)
version: 1.0.0
status: draft
origin: reverse-engineered
confidence: high
owner: pendiente-asignacion
last_review: 2026-07-15
review_cycle_days: 180
audience: [dev, ops, integradores, agentes-automaticos]
classification: uso-interno
traces: [ADR-0003, ADR-0004, ADR-0005]
supersedes: null
---

# IAConnect.API — Web API REST (servicio)

> **Resumen ejecutivo.** Es el único ejecutable de servidor: expone la API REST, compone las capas (DI), aplica
> autenticación JWT, autorización por rol y por tenant, Swagger, CORS y health checks. Delega toda la lógica en
> `IAConnect.Application`. `type: service` (Marco §7).

## Tecnología y dependencias

- **.NET 8** (`Microsoft.NET.Sdk.Web`), C# con `Nullable` e `ImplicitUsings` habilitados, `GenerateDocumentationFile`.
- Paquetes: `Microsoft.AspNetCore.Authentication.JwtBearer`, `Microsoft.AspNetCore.OpenApi`, `Swashbuckle.AspNetCore 6.9`.
- Referencias: `IAConnect.Application`, `IAConnect.Infrastructure`, `IAConnect.Domain`.
- `InternalsVisibleTo("IAConnect.Tests")` y `public partial class Program {}` para tests de integración.

## Superficie REST

Ver [catálogo](../../05-apis/catalog.md) + [`openapi.yaml`](../../05-apis/openapi.yaml). Controladores:
`AuthController` (`/api/auth`), `AIController` (`/api/ai/{tenantId}`), `TenantsController` (`/api/tenants`),
`KnowledgeController` (`/api/tenants/{tenantId}/knowledge`).

## Pipeline y composición (`Program.cs`)

```text
GlobalExceptionMiddleware → Swagger/SwaggerUI → CORS → Authentication(JWT) → Authorization
  → TenantResolverMiddleware → MapControllers → /health → GET /
```

Registro DI (todo `Scoped` salvo indicado):
- **DataManagers** (7): `ILutTenantsDataManager`, `ISysSesiones/…/SysRefreshTokensDataManager`.
- **Servicios** (Application): Auth, Chat, Completion, Analyze, Summarize, Improve, Tenant, PromptBuilder, RAGEngine, Knowledge, ImageValidator.
- **`IAIProviderFactory`** → `AIProviderFactory` (**Singleton**).
- **`HttpClient` "Claude"** → `https://api.anthropic.com/`, timeout 60 s.
- `DataEntityCore.Configure(ConnectionStrings:IAConnect)` al arranque.

## Middleware y filtros

| Componente | Comportamiento | Fuente |
|---|---|---|
| `GlobalExceptionMiddleware` | Mapea excepciones de dominio → HTTP (`404/401/423/400/502/500`), respuesta `{error, statusCode}`, log 5xx=Error | `Middleware/GlobalExceptionMiddleware.cs` |
| `TenantResolverMiddleware` | Resuelve tenant de la ruta; 404 si inexistente/inactivo; guarda en `HttpContext.Items["Tenant"]` | `Middleware/TenantResolverMiddleware.cs` |
| `TenantAccessFilter` | Autoriza acceso: admin→cualquier tenant; operador→su `id_tenant`; si no, 403 | `Middleware/TenantAccessFilter.cs` |

## Autenticación y autorización

JWT Bearer (issuer/audience/lifetime/firma HmacSha256, `ClockSkew=0`). Roles `admin`/`operador`.
`/api/ai/{tenantId}` requiere JWT + `TenantAccessFilter`. Tenants/Knowledge/CRUD usuarios requieren rol `admin`.
Detalle → [06-crosscutting](../../01-architecture/06-crosscutting.md).

## Configuración (claves, sin valores/secretos)

`ConnectionStrings:IAConnect` · `Jwt:{SecretKey,Issuer,Audience,AccessTokenExpirationMinutes,RefreshTokenExpirationDays}`
· `Cors:AllowedOrigins[]` · `Encryption:AesKey` (⚠ no usado — ver crosscutting) · `AIProviders:{Gemini,Claude,OpenAI}:{ApiKey,DefaultModel}` (⚠ `DefaultModel` no consumido).
Variable de entorno adicional: `IACONNECT_ENCRYPTION_KEY` (cifrado de API keys de tenant).

## Cómo ejecutar

```bash
# Local
dotnet run --project IAConnect.API        # http://localhost:5051  (Swagger en /swagger)
# Contenedor
docker compose up                          # API :8080 + SQL Server :1433
```
Requiere `ConnectionStrings:IAConnect` apuntando a una instancia con el esquema de `scripts/01_create_database.sql`.

## Runbook

Operación, salud y diagnóstico → [07-operations/runbook-api.md](../../07-operations/runbook-api.md).

## Gaps / observaciones

- Swagger habilitado en **todos** los entornos (revisar exposición en producción).
- Orden de `TenantResolverMiddleware` vs. enrutamiento a verificar (ver [componentes](../../01-architecture/03-components.md)).
- Sin versionado de URL, sin correlación de requests, formato de error no estándar (RFC 9457).
- Sin `CHANGELOG.md` en el origen → `GAP-CHANGELOG`.
