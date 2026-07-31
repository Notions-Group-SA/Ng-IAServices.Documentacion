> **Índice de pruebas y DevOps.** Estrategia de test, contenedores, configuración y despliegue.
> Fuentes: `IAConnect.Tests/**`, `Dockerfile`, `docker-compose.yml`, `appsettings*.json`, `scripts/`.

# 06 · Pruebas y DevOps — IAConnect

## Pruebas (`IAConnect.Tests`, xUnit)

| Tipo | Archivos | Cubre |
|---|---|---|
| Unit — Services | `Unit/Services/*Tests.cs` (Auth, Chat, Completion, Analyze, Summarize, Improve, Tenant, RAGEngine, PromptBuilder, ImageValidator) | Lógica de aplicación |
| Unit — Providers | `Unit/Providers/AIProviderFactoryTests.cs` | Selección de proveedor |
| Unit — Middleware | `Unit/Middleware/TenantResolverMiddlewareTests.cs` | Resolución de tenant |
| Integration | `Integration/*` (Auth, HealthCheck, MultiTenantIsolation, Tenants + `IAConnectWebApplicationFactory`) | API end-to-end |
| Helpers | `Helpers/{MockDataHelper,TestJwtHelper}.cs` | Datos y JWT de prueba |

`IAConnect.API` expone `public partial class Program {}` para `WebApplicationFactory` (tests de integración).
`InternalsVisibleTo("IAConnect.Tests")` declarado en `IAConnect.API.csproj`.

## Contenedores

- **`Dockerfile`** (multi-stage): `sdk:8.0` build/publish → `aspnet:8.0` runtime; usuario **no-root** (`appuser`);
  `EXPOSE 8080`; `HEALTHCHECK` a `/health`; `ENTRYPOINT dotnet IAConnect.API.dll`.
- **`docker-compose.yml`**: servicios `iaconnect-api` (8080) + `sqlserver` (`mssql/server:2022`, 1433) con
  healthchecks y volumen persistente; secretos por variables de entorno (`JWT_SECRET_KEY`, `SA_PASSWORD`, …).

## Configuración (claves, sin secretos)

| Sección | Claves |
|---|---|
| `ConnectionStrings` | `IAConnect` (cadena SQL Server; **vacía en `appsettings.json`**) |
| `Jwt` | `SecretKey`, `Issuer`, `Audience`, `AccessTokenExpirationMinutes`, `RefreshTokenExpirationDays` |
| `Cors` | `AllowedOrigins[]` |
| `Encryption` | `AesKey` |
| `AIProviders` | `Gemini/Claude/OpenAI` → `{ ApiKey, DefaultModel }` |

## Base de datos / despliegue

- Script de creación: `scripts/01_create_database.sql` (BD, 7 tablas, índices, stored procedures CRUD).
- Ejecución vía `sqlcmd`. **El encabezado del script trae credenciales de ejemplo que no se reproducen aquí.**
- Utilidades: `_hashgen/` (generación de hashes de contraseña de usuario para seed).

## Docs de origen relacionadas

`docs/07_calidad_y_pruebas/{estrategia-calidad,plan-pruebas}_v1.0.md` ·
`docs/08_devops/{deployment-config,dockerfile-spec}_v1.0.md` · `docs/guia-configuracion-pruebas.md`.

## Detalle

Doc generada: `../IAConnect-docs/docs/07-operations/`, `docs/08-onboarding/developer-setup.md`,
`docs/pieces/IAConnect.Tests/`.
