---
doc_id: PIECE-TESTS-001
doc_type: piece-readme
title: "IAConnect.Tests — Pruebas"
version: 1.0.0
status: draft
origin: reverse-engineered
confidence: high
owner: pendiente-asignacion
last_review: 2026-07-15
review_cycle_days: 180
audience: [dev, qa, agentes-automaticos]
classification: uso-interno
traces: []
supersedes: null
---

# IAConnect.Tests — Pruebas

## 1. Resumen ejecutivo

`IAConnect.Tests` (`NG/Ng-IAServices/IAConnect.Tests/IAConnect.Tests.csproj`) es el proyecto de pruebas
automatizadas de la solución IAConnect. Contiene **17 clases de test** (12 unitarias + 5 de integración,
más 2 helpers de soporte) que cubren la capa de aplicación (`IAConnect.Application`), un middleware y un
provider factory de infraestructura, y el pipeline HTTP completo de `IAConnect.API` (auth, autorización
multi-tenant, tenants, health check).

El proyecto referencia directamente los cuatro proyectos productivos (`IAConnect.API`, `IAConnect.Application`,
`IAConnect.Domain`, `IAConnect.Infrastructure` — `IAConnect.Tests.csproj#L26-L30`) y **no ejecuta contra una
base de datos real**: todos los `DataManager` (acceso a SQL Server vía stored procedures) se sustituyen por
mocks de Moq, tanto en tests unitarios como de integración. Es, por tanto, un suite de pruebas de lógica de
negocio y de contrato HTTP, no de persistencia real.

## 2. Estrategia: unit vs integration

| Aspecto | Unit (`Unit/`) | Integration (`Integration/`) |
|---|---|---|
| Objetivo | Verificar una clase aislada (servicio, middleware, factory) | Verificar el pipeline HTTP completo: routing, auth JWT, filtros de autorización, serialización |
| Cómo se instancia el SUT | `new ServicioX(mock1.Object, mock2.Object, ...)` directo | `IAConnectWebApplicationFactory : WebApplicationFactory<Program>` + `HttpClient` real sobre servidor en memoria |
| Dependencias externas | Mockeadas una a una con `Moq` | Mockeadas centralizadamente en la factory (mismos mocks para toda la suite de esa clase) |
| Base de datos | No (mock de `I*DataManager`) | No (mock de `I*DataManager`, ver §4) |
| Aserciones | `FluentAssertions` (`.Should()...`) | `FluentAssertions` sobre `HttpResponseMessage.StatusCode` / contenido |

Framework y librerías, según `IAConnect.Tests.csproj#L12-L20`:

| Paquete | Versión | Rol |
|---|---|---|
| `xunit` / `xunit.runner.visualstudio` | 2.5.3 | Framework de test (`[Fact]`, `[Theory]`, `[InlineData]`) |
| `Moq` | 4.20.72 | Mocking de interfaces (`I*DataManager`, `IAIProvider`, `IAIProviderFactory`, `IPromptBuilder`, `IRAGEngine`, `IImageValidator`) |
| `FluentAssertions` | 8.9.0 | Aserciones fluidas (`result.Should().NotBeNull()`, etc.) |
| `Microsoft.AspNetCore.Mvc.Testing` | 8.0.0 | `WebApplicationFactory<Program>` para tests de integración in-memory |
| `Microsoft.NET.Test.Sdk` | 17.8.0 | SDK de ejecución de tests .NET |
| `coverlet.collector` | 6.0.0 | Recolección de cobertura de código (`dotnet test --collect:"XPlat Code Coverage"`) |

No hay librerías de contract-testing (Pact), de generación de datos (Bogus/AutoFixture) ni de snapshot testing;
los datos de prueba se construyen a mano vía `Helpers/MockDataHelper.cs`.

## 3. Inventario de pruebas

| Archivo | Tipo | Qué verifica | Clase/componente bajo prueba |
|---|---|---|---|
| `Unit/Middleware/TenantResolverMiddlewareTests.cs` | unit | Resuelve el tenant activo desde la ruta y lo inyecta en `HttpContext.Items["Tenant"]`; 404 si no existe o está inactivo; pass-through si no hay `tenantId` en la ruta | `TenantResolverMiddleware` (`IAConnect.API.Middleware`) |
| `Unit/Providers/AIProviderFactoryTests.cs` | unit | Selección de provider según `Tenant.ProveedorIA` (`gemini`/`claude`/`openai`); error si falta `IACONNECT_ENCRYPTION_KEY`; `ArgumentException` si el proveedor es desconocido | `AIProviderFactory` (`IAConnect.Infrastructure.Providers`) |
| `Unit/Services/AnalyzeServiceTests.cs` | unit | Análisis con los 3 `TipoAnalisis` (`sentiment`/`classification`/`entities`); tipo inválido → `ArgumentException`; tenant inexistente → `TenantNotFoundException` | `AnalyzeService` |
| `Unit/Services/AuthServiceTests.cs` | unit | Login válido (JWT + refresh token), credenciales inválidas, cuenta bloqueada, bloqueo automático al 5º intento fallido, refresh de token (válido/revocado/expirado), alta de usuario con hash bcrypt, baja lógica | `AuthService` |
| `Unit/Services/ChatServiceTests.cs` | unit | Chat con sesión nueva (persiste user+assistant), sesión existente (carga historial), envío de imagen (valida vía `IImageValidator`), imagen no permitida, tenant inexistente | `ChatService` |
| `Unit/Services/CompletionServiceTests.cs` | unit | Completion básico con `PromptBuilder`+`RAGEngine`+provider; tenant inexistente | `CompletionService` |
| `Unit/Services/ImageValidatorTests.cs` | unit | Imagen válida (PNG/JPG), tenant con imágenes deshabilitadas, tamaño excede `MaxTamanoImagenKB`, formato no permitido, tenant inexistente | `ImageValidator` |
| `Unit/Services/ImproveServiceTests.cs` | unit | Mejora de texto con los 4 `ObjetivoMejora` (`clarity`/`formality`/`brevity`/`expand`); objetivo inválido; tenant inexistente | `ImproveService` |
| `Unit/Services/PromptBuilderTests.cs` | unit | Ensamblado del system prompt: sin RAG/historial, con fragmentos RAG, con historial de conversación, orden de secciones (system → RAG → historial → consulta), omisión de sección RAG vacía | `PromptBuilder` |
| `Unit/Services/RAGEngineTests.cs` | unit | Búsqueda de fragmentos relevantes por *keyword matching* (fallback sin embeddings), respeto del `topK`, sin fragmentos, sin coincidencias, tenant inexistente | `RAGEngine` |
| `Unit/Services/SummarizeServiceTests.cs` | unit | Resumen de documento con provider y métricas de tokens; tenant inexistente | `SummarizeService` |
| `Unit/Services/TenantServiceTests.cs` | unit | Listado y mapeo a DTO, alta con cifrado AES de `ApiKeyIA`, actualización parcial (solo campos provistos), baja lógica (`Activo=false`), tenant inexistente en cada operación | `TenantService` |
| `Integration/AuthControllerIntegrationTests.cs` | integration | `POST /api/auth/login` con credenciales válidas (200) e inválidas (401, vía `GlobalExceptionMiddleware` mapeando `InvalidCredentialsException`); `GET /api/auth/usuarios` con token admin (200) y sin token (401) | `AuthController` end-to-end |
| `Integration/HealthCheckIntegrationTests.cs` | integration | `GET /health` (200); `GET /` devuelve 200 y contiene `"Running"` | Endpoint de health check / `Program` |
| `Integration/MultiTenantIsolationTests.cs` | integration | Admin puede operar sobre cualquier `{tenantId}` de ruta; operador con tenant distinto al de su JWT recibe 403; operador con su propio tenant no recibe 403 | `TenantAccessFilter` + `AIController` (endpoint `/api/ai/{tenantId}/completion`) |
| `Integration/TenantsControllerIntegrationTests.cs` | integration | `GET /api/tenants` con token admin (200), con token operador (403), sin token (401) | `TenantsController` |
| `Integration/IAConnectWebApplicationFactory.cs` | infraestructura de test (no es un caso de prueba) | Host in-memory de `IAConnect.API` con todos los `DataManager` reemplazados por `Mock<T>` públicos | `WebApplicationFactory<Program>` |
| `Helpers/MockDataHelper.cs` | helper | Fábricas de entidades de prueba: `Tenant`, `Usuario` (hash bcrypt `workFactor:4`), `Sesion`, `RefreshToken`, `FragmentoConocimiento`, `AIResponse` | — |
| `Helpers/TestJwtHelper.cs` | helper | Genera JWT HS256 de prueba con claims `sub`, `nombre_usuario`, `id_tenant`, `rol`/`ClaimTypes.Role`, firmados con una clave simétrica de prueba fija (no productiva) | — |

## 4. Cómo se levantan las pruebas de integración

`Integration/IAConnectWebApplicationFactory.cs` extiende `WebApplicationFactory<Program>`
(`Microsoft.AspNetCore.Mvc.Testing`) y arranca `IAConnect.API` completo en memoria (`TestServer`), sin proceso
ni puerto real:

- **Entorno**: `builder.UseEnvironment("Testing")` (`IAConnectWebApplicationFactory.cs#L27`).
- **Configuración JWT in-memory**: inyecta `Jwt:SecretKey`/`Issuer`/`Audience`/expiraciones vía
  `ConfigureAppConfiguration` + `AddInMemoryCollection` (`IAConnectWebApplicationFactory.cs#L29-L39`), usando la
  misma clave de prueba (`TestJwtKey`) con la que `Helpers/TestJwtHelper.cs` firma los tokens de prueba —
  **no es un secreto productivo**.
- **Base de datos: no hay ninguna.** `ConfigureServices` reemplaza (`ReplaceService<T>`,
  `IAConnectWebApplicationFactory.cs#L54-L59`) los siete `DataManager` de `IAConnect.Infrastructure`
  (`ILutTenantsDataManager`, `ISysUsuariosDataManager`, `ISysSesionesDataManager`, `ISysMensajesDataManager`,
  `ISysMetricasUsoDataManager`, `ISysFragmentosConocimientoDataManager`, `ISysRefreshTokensDataManager`) por
  `Mock<T>.Object`, expuestos como propiedades públicas de la factory (`TenantsDm`, `UsuariosDm`, etc.). No se usa
  Testcontainers, SQL LocalDB ni una cadena de conexión real: cada test configura `.Setup(...)` sobre estos mocks
  antes de invocar el endpoint.
- **Autenticación en el cliente**: los tests generan un JWT con `TestJwtHelper.GenerateToken(role:, tenantId:)`
  y lo setean en `HttpClient.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", token)`.
- **Ciclo de vida**: cada clase implementa `IClassFixture<IAConnectWebApplicationFactory>`, por lo que la
  factory (y sus mocks) se comparte entre los tests de esa clase; cada test re-configura el `.Setup()` que
  necesita antes del `Act`.

En consecuencia, las pruebas de integración validan el **pipeline HTTP real** (routing, `TenantResolverMiddleware`,
autenticación JWT, `TenantAccessFilter`, `GlobalExceptionMiddleware`, serialización JSON) pero **no** validan
las stored procedures ni el esquema real de SQL Server — ver gap en §6, consistente con `GAP-DB-LIVE` de
`docs-manifest.yaml`.

## 5. Cómo ejecutar

```bash
# Todo el suite, desde la carpeta del proyecto de tests o de la solución
dotnet test NG/Ng-IAServices/IAConnect.Tests/IAConnect.Tests.csproj

# Solo unitarios
dotnet test IAConnect.Tests.csproj --filter "FullyQualifiedName~Unit"

# Solo integración
dotnet test IAConnect.Tests.csproj --filter "FullyQualifiedName~Integration"

# Con cobertura (coverlet.collector ya referenciado en el csproj)
dotnet test IAConnect.Tests.csproj --collect:"XPlat Code Coverage"
```

No requiere variables de entorno ni servicios externos: las claves JWT y de cifrado (`IACONNECT_ENCRYPTION_KEY`,
`Jwt__SecretKey`) que necesitan algunos tests se setean en el propio código de test (`Environment.SetEnvironmentVariable`)
con valores fijos de prueba, no productivos.

## 6. Cobertura observada y gaps

### Cubierto

| Área | Evidencia |
|---|---|
| Servicios de `IAConnect.Application` (Analyze, Auth, Chat, Completion, ImageValidator, Improve, PromptBuilder, RAGEngine, Summarize, Tenant) | 1 clase de test unitario por servicio, `Unit/Services/*.cs` |
| `TenantResolverMiddleware` | `Unit/Middleware/TenantResolverMiddlewareTests.cs` |
| `AIProviderFactory` (selección de proveedor) | `Unit/Providers/AIProviderFactoryTests.cs` |
| `TenantAccessFilter` (403 cross-tenant) | indirecto, vía `Integration/MultiTenantIsolationTests.cs` y `TenantsControllerIntegrationTests.cs` (`TenantAccessFilter.cs#L37-L44`) |
| `GlobalExceptionMiddleware` (mapeo de excepciones a HTTP status) | indirecto, solo el caso `InvalidCredentialsException → 401` vía `AuthControllerIntegrationTests.cs` (`GlobalExceptionMiddleware.cs#L35`) |
| `AuthController` (`/login`, `/usuarios`) y `TenantsController` (`/api/tenants`) | `Integration/AuthControllerIntegrationTests.cs`, `Integration/TenantsControllerIntegrationTests.cs` |
| `AIController` — solo endpoint `/completion` | `Integration/MultiTenantIsolationTests.cs` (como vehículo para probar autorización, no cubre el resto de casos de negocio de completion) |
| Health check (`/health`, `/`) | `Integration/HealthCheckIntegrationTests.cs` |

### Gaps identificados

| Gap | Detalle |
|---|---|
| **`KnowledgeService` / `KnowledgeController` sin tests** | No existe `Unit/Services/KnowledgeServiceTests.cs` ni `Integration/KnowledgeControllerIntegrationTests.cs`, pese a que `IAConnect.Application/Services/KnowledgeService.cs` e `IAConnect.API/Controllers/KnowledgeController.cs` existen en el origen. La carga/gestión de la base de conocimiento (fragmentos + embeddings) usada por `RAGEngine` **no tiene cobertura**. |
| **Providers concretos sin tests unitarios propios** | `AIProviderFactoryTests.cs` solo prueba la lógica de *selección* de provider; `ClaudeProvider`, `GeminiProvider` y `OpenAIProvider` (`IAConnect.Infrastructure/Providers/*.cs`) — construcción de request HTTP, parseo de respuesta, manejo de errores del proveedor externo — **no tienen tests dedicados**. |
| **Endpoints de `AIController` incompletos a nivel integración** | Solo `/completion` se ejercita en un test de integración (y únicamente para verificar autorización, no el contrato de respuesta). `/chat`, `/analyze`, `/summarize`, `/improve` solo están probados a nivel de servicio unitario (mockeado), no como pipeline HTTP end-to-end. |
| **`GlobalExceptionMiddleware` parcialmente cubierto** | Solo se verifica el mapeo de `InvalidCredentialsException → 401`. No hay evidencia de tests para el mapeo de `TenantNotFoundException`, `ImageNotAllowedException`, `AccountLockedException` u otras excepciones de dominio a sus códigos HTTP correspondientes. |
| **Sin pruebas contra base de datos real** | Todos los `I*DataManager` (`IAConnect.Infrastructure/DataManagers/*`) se mockean con Moq en unit e integration tests (§4). No hay tests de las stored procedures ni del esquema SQL Server real — gap ya declarado como `GAP-DB-LIVE` en `docs-manifest.yaml`. Sin introspección en vivo de la base, la [matriz de casos de prueba derivada del modelo de datos](../../03-data/test-data-matrix.md) (identificadores `TC-*`) no puede contrastarse contra ejecuciones reales de este suite; a la fecha de este relevamiento ese archivo no existía todavía en `docs/03-data/`. |
| **Sin tests de `PromptBuilder`/`RAGEngine` con embeddings reales** | `RAGEngineTests.cs` solo ejercita el *fallback* por coincidencia de palabras clave (`VectorEmbedding = null` en todos los fixtures); no hay caso que ejercite la ruta de búsqueda por similitud vectorial. |
| **Sin pruebas de carga, contrato (OpenAPI) ni seguridad automatizadas** | El suite no incluye smoke tests de OpenAPI, fuzzing de payloads, ni pruebas de límites de tasa/tamaño más allá de `ImageValidatorTests`. |

> Nota de trazabilidad: los JWT y claves de prueba citados en este README (`TestJwtHelper.cs`, `IAConnectWebApplicationFactory.TestJwtKey`) son valores fijos de **solo prueba**, sin valor productivo; no se reproducen aquí literalmente por buena práctica aunque no constituyen un secreto real.
