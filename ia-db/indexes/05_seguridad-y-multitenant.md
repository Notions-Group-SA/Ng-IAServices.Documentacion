> **Índice de seguridad y multi-tenancy.** Autenticación JWT, refresh tokens, aislamiento por tenant y
> autorización. Fuentes: `API/Program.cs`, `API/Middleware/**`, `Application/Services/AuthService.cs`.

# 05 · Seguridad y Multi-tenant — IAConnect

## Autenticación (JWT Bearer)

- Configurada en `Program.cs` (`AddAuthentication(JwtBearer)`): valida issuer, audience, lifetime y firma
  (`SymmetricSecurityKey` desde `Jwt:SecretKey`), `ClockSkew = 0`.
- Config `Jwt`: `SecretKey`, `Issuer` (`IAConnect`), `Audience` (`IAConnect.API`),
  `AccessTokenExpirationMinutes` (60), `RefreshTokenExpirationDays` (7). Fuente: `appsettings.json`.
- Login (`AuthService.LoginAsync`) verifica credenciales (**hash BCrypt**) contra `sys_Usuarios.Password_Hash`, emite
  JWT (**HmacSha256**) + refresh token (64 bytes, rotación), y aplica **bloqueo a los 5 intentos fallidos durante 15
  min** (`Intentos_Fallidos`, `Fecha_Bloqueo` → `AccountLockedException` → **423**).
- Refresh: `sys_Refresh_Tokens` (token único, `Revocado`, `Fecha_Expiracion`). Logout revoca el refresh token.

## Autorización

| Nivel | Mecanismo | Dónde |
|---|---|---|
| Autenticado | `[Authorize]` | AIController, logout |
| Por rol | `[Authorize(Roles="admin")]` | TenantsController, KnowledgeController, CRUD usuarios |
| Por tenant | `TenantAccessFilter` (`ServiceFilter`) | AIController (`/api/ai/{tenantId}`) |
| Roles | `admin`, `operador` (`sys_Usuarios.Rol` con CHECK) | — |

## Multi-tenancy

```mermaid
flowchart LR
    Req[Request /api/ai/{tenantId}/...] --> Auth[JWT auth]
    Auth --> Resolver[TenantResolverMiddleware: resuelve tenant del contexto]
    Resolver --> Filter[TenantAccessFilter: ¿el usuario puede acceder a {tenantId}?]
    Filter -->|sí| Ctrl[Controller]
    Filter -->|no| F403[403 Forbidden]
```

- **Resolución:** `TenantResolverMiddleware` (registrado después de auth) resuelve el tenant.
- **Aislamiento:** el `{tenantId}` es parte de la ruta; `TenantAccessFilter` valida que el usuario autenticado
  tenga acceso a ese tenant. Datos particionados por `Id_Tenant` en todas las tablas de negocio.
- **Configuración por tenant:** proveedor IA, modelo, system prompt, temperatura, límites de tokens e imagen,
  expiración de tokens y `ApiKey_IA` viven en `lut_Tenants`.
- Pruebas de aislamiento: `IAConnect.Tests/Integration/MultiTenantIsolationTests.cs`.

## Manejo de errores

`GlobalExceptionMiddleware` centraliza el mapeo excepción→HTTP: `InvalidCredentials/AccountLocked`→401,
`TenantNotFound`→404, `ImageNotAllowed`→400, `ProviderUnavailable`→502/503 (verificar código exacto en la fuente).

## ⚠ Observaciones de seguridad (para revisión humana)

- `Jwt:SecretKey` y `Encryption:AesKey` en `appsettings.json` están **vacíos**; en `docker-compose.yml` y
  `appsettings.Development.json` hay **valores de desarrollo por defecto** ("dev-secret-key…") que **no deben usarse
  en producción** — deben venir de variables de entorno / gestor de secretos.
- `lut_Tenants.ApiKey_IA` guarda la clave del proveedor **en la BD cifrada AES** (clave por env
  `IACONNECT_ENCRYPTION_KEY`); ⚠ **fallback a texto plano si la env var falta** (`GAP-ENC-FALLBACK`). `Encryption:AesKey`
  de `appsettings.json` está muerto.
- CORS: orígenes permitidos vía `Cors:AllowedOrigins` (default `http://localhost:3000`).

## Detalle

Doc de origen: `docs/05_arquitectura_tecnica/{autenticacion,multitenant-spec}_v1.0.md`.
Doc generada: `../IAConnect-docs/docs/01-architecture/06-crosscutting.md` y `docs/pieces/IAConnect.API/`.
