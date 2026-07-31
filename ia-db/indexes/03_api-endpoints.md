> **Índice de la API REST.** Controladores, endpoints, autorización y DTOs. Fuente primaria:
> `IAConnect.API/Controllers/**`, `IAConnect.Application/DTOs/**`. Base path: `/api`.

# 03 · API Endpoints — IAConnect

## Controladores y rutas

| Controlador | Ruta base | Autorización | Endpoints |
|---|---|---|---|
| `AuthController` | `/api/auth` | mixta | login, refresh, logout, CRUD usuarios |
| `AIController` | `/api/ai/{tenantId}` | `[Authorize]` + `TenantAccessFilter` | chat, completion, analyze, summarize, improve |
| `TenantsController` | `/api/tenants` | `[Authorize(Roles=admin)]` | CRUD tenants |
| `KnowledgeController` | `/api/tenants/{tenantId}/knowledge` | `[Authorize(Roles=admin)]` | subir documento, listar fragmentos |

## Catálogo de endpoints

| Método | Ruta | Auth | Request DTO | Respuesta |
|---|---|---|---|---|
| POST | `/api/auth/login` | anónimo | `LoginRequestDto` | `LoginResponseDto` (JWT + refresh) |
| POST | `/api/auth/refresh` | anónimo | `RefreshTokenRequestDto` | `LoginResponseDto` |
| POST | `/api/auth/logout` | JWT | `RefreshTokenRequestDto` | 204 |
| GET | `/api/auth/usuarios` | admin | — | `UsuarioDto[]` |
| POST | `/api/auth/usuarios` | admin | `CreateUsuarioRequestDto` | 201 `UsuarioDto` |
| GET | `/api/auth/usuarios/{id}` | admin | — | `UsuarioDto` |
| PUT | `/api/auth/usuarios/{id}` | admin | `UpdateUsuarioRequestDto` | `UsuarioDto` |
| DELETE | `/api/auth/usuarios/{id}` | admin | — | 204 (soft-delete) |
| POST | `/api/ai/{tenantId}/chat` | JWT+tenant | `ChatRequestDto` | `AIResponseDto` |
| POST | `/api/ai/{tenantId}/completion` | JWT+tenant | `CompletionRequestDto` | `AIResponseDto` |
| POST | `/api/ai/{tenantId}/analyze` | JWT+tenant | `AnalysisRequestDto` | `AnalysisResponseDto` |
| POST | `/api/ai/{tenantId}/summarize` | JWT+tenant | `SummarizeRequestDto` | `SummarizeResponseDto` |
| POST | `/api/ai/{tenantId}/improve` | JWT+tenant | `ImproveRequestDto` | `ImproveResponseDto` |
| GET | `/api/tenants` | admin | — | `TenantDto[]` |
| POST | `/api/tenants` | admin | `CreateTenantRequestDto` | 201 `TenantDto` |
| GET | `/api/tenants/{tenantId}` | admin | — | `TenantDto` |
| PUT | `/api/tenants/{tenantId}` | admin | `UpdateTenantRequestDto` | `TenantDto` |
| DELETE | `/api/tenants/{tenantId}` | admin | — | 204 |
| POST | `/api/tenants/{tenantId}/knowledge` | admin | `multipart/form-data` (IFormFile) | `{ tenantId, fileName, chunksCreated }` |
| GET | `/api/tenants/{tenantId}/knowledge` | admin | — | fragmentos `[{Id,DocumentoOrigen,IndiceFragmento,Contenido,FechaAlta}]` |
| GET | `/health` | anónimo | — | health check |
| GET | `/` | anónimo | — | `{ Status, Service, Version }` |

## Códigos de estado usados

`200` OK · `201` Created · `204` NoContent · `400` datos inválidos · `401` no autenticado / credenciales /
refresh inválido · `403` sin acceso al tenant o rol insuficiente · `404` tenant/usuario no encontrado.
Errores traducidos por `GlobalExceptionMiddleware`.

## DTOs (`IAConnect.Application/DTOs/`)

- **Requests:** Analysis, Chat, Completion, CreateTenant, CreateUsuario, Improve, Login, RefreshToken,
  Summarize, UpdateTenant, UpdateUsuario.
- **Responses:** AIResponse, AnalysisResponse, ImproveResponse, LoginResponse, SummarizeResponse, Tenant, Usuario.

## Notas de contrato

- La API expone **Swagger UI en todos los entornos** (`app.UseSwagger()/UseSwaggerUI()`), con seguridad Bearer JWT.
- No existe un `openapi.yaml` versionado en el origen → se **infiere** en `../IAConnect-docs/docs/05-apis/openapi.yaml`
  (Marco §6); divergencias contrato↔código se reportan, no se ajustan en silencio.
- `AIController` obtiene el `userId` del claim `NameIdentifier`/`sub` del JWT.

## Detalle

Catálogo + OpenAPI inferido: `../IAConnect-docs/docs/05-apis/`. Doc de origen:
`docs/05_arquitectura_tecnica/api-rest-spec_v1.0.md`.
