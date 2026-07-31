---
doc_id: ONB-SETUP-001
doc_type: developer-setup
title: Puesta a punto del desarrollador — IAConnect
version: 1.0.0
status: draft
origin: reverse-engineered
confidence: high
owner: pendiente-asignacion
last_review: 2026-07-15
review_cycle_days: 180
audience: [dev, agentes-automaticos]
classification: uso-interno
traces: []
supersedes: null
---

# Puesta a punto del desarrollador — IAConnect

> **Resumen ejecutivo.** Cómo levantar la solución localmente desde cero. Meta: un dev nuevo corre la API + BD y
> ve Swagger en minutos, con datos sintéticos.

## Requisitos

- **.NET 8 SDK**, Docker (para SQL Server), un cliente SQL (sqlcmd/Azure Data Studio).
- Claves de proveedor IA de **prueba** (Gemini/Claude/OpenAI) si se quieren probar los endpoints IA (opcional).

## 1. Clonar y restaurar

```bash
git clone <repo> Ng-IAServices && cd Ng-IAServices
dotnet restore IAConnect.slnx
```

## 2. Base de datos

```bash
# Opción A — todo con compose (API + SQL Server)
docker compose up -d sqlserver
sqlcmd -S localhost -U sa -P <sa-password-del-entorno> -i scripts/01_create_database.sql
```
> Usá una contraseña propia por variable de entorno; **no** reutilices los defaults de ejemplo del compose.

## 3. Configuración (sin secretos en el repo)

Definí por *user-secrets* o variables de entorno (no en `appsettings.json`):

| Clave | Para qué |
|---|---|
| `ConnectionStrings:IAConnect` | Cadena a SQL Server local |
| `Jwt:SecretKey` | Firma JWT (≥32 chars) |
| `IACONNECT_ENCRYPTION_KEY` | Cifrado AES de la API key de tenant (⚠ sin ella, la key se guardaría en claro) |
| `AIProviders:*:ApiKey` | (Opcional) claves de proveedor para pruebas |

```bash
cd IAConnect.API
dotnet user-secrets set "ConnectionStrings:IAConnect" "Server=localhost;Database=IAConnect;User Id=sa;Password=<tu-pass>;TrustServerCertificate=true;"
dotnet user-secrets set "Jwt:SecretKey" "<clave-larga-de-desarrollo>"
```

## 4. Ejecutar

```bash
dotnet run --project IAConnect.API     # http://localhost:5051 → /swagger
# Web demo (Blazor Server)
dotnet run --project IAConnect.Demo.Web
```

## 5. Crear datos de prueba

- Un usuario admin: usar `_hashgen/` para generar el `Password_Hash` (BCrypt) e insertarlo en `sys_Usuarios`, o el
  seed sintético de [`fixtures/iaconnect.seed.yaml`](../03-data/fixtures/iaconnect.seed.yaml).
- Un tenant: `POST /api/tenants` (requiere JWT admin) con proveedor/modelo/clave de prueba.

## 6. Pruebas

```bash
dotnet test IAConnect.Tests    # xUnit; integración in-memory (WebApplicationFactory), sin BD real
```

## Navegación de la documentación

- Arquitectura → [01-architecture](../01-architecture/01-context.md) · API → [05-apis](../05-apis/catalog.md)
  · Datos → [03-data](../03-data/data-dictionary.md) · Piezas → [pieces/](../pieces/).
- Contexto rápido para agentes → [ia-db](../../../ia-db/README.md).
