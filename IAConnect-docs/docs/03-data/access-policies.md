---
doc_id: DATA-ACCESS-001
doc_type: access-policies
title: "Políticas de acceso a datos — IAConnect"
version: 1.0.0
status: draft
origin: reverse-engineered
confidence: high
owner: pendiente-asignacion
last_review: 2026-07-15
review_cycle_days: 120
audience: [dev, dba, qa, auditoria, agentes-automaticos]
classification: uso-interno
sources:
  - NG/Ng-IAServices/scripts/01_create_database.sql
  - NG/Ng-IAServices/docker-compose.yml
  - NG/Ng-IAServices/IAConnect.API/appsettings.json
  - NG/Ng-IAServices/docs/05_arquitectura_tecnica/autenticacion_v1.0.md
  - NG/Ng-IAServices/docs/05_arquitectura_tecnica/multitenant-spec_v1.0.md
related:
  - ./data-dictionary.md
  - ./er-diagrams/iaconnect.dbml
---

# Políticas de acceso a datos — IAConnect

> **Origen:** ingeniería inversa. **Este documento NO contiene credenciales, cadenas de conexión reales, API keys ni contraseñas.** Todo mecanismo se describe sin valores.

## 1. Modelo de acceso (dos capas)

IAConnect separa la autorización en dos planos:

1. **Plano de aplicación (roles de negocio).** La API (.NET 8) autentica con JWT y autoriza por rol y por tenant. Es donde vive la política funcional.
2. **Plano de base de datos.** La aplicación se conecta con **una única cuenta de servicio SQL** y ejecuta exclusivamente **stored procedures**. No hay usuarios de BD por rol de negocio; la BD no distingue `admin` de `operador`.

## 2. Roles de negocio → permisos (plano de aplicación)

| Rol | Alcance | Permisos sobre datos (efectivos vía API) |
|---|---|---|
| `admin` (global, `Id_Tenant` NULL) | Toda la plataforma | Alta/baja/modificación de tenants (`lut_Tenants`), gestión de usuarios (`sys_Usuarios`), lectura de métricas y conocimiento de cualquier tenant. |
| `admin` (de tenant) | Su `Id_Tenant` | Gestión de la configuración y usuarios de su tenant; lectura de sus sesiones/mensajes/métricas/fragmentos. |
| `operador` | Su `Id_Tenant` | Operación del asistente: crear sesiones y mensajes, consultar historial y conocimiento de su tenant. Sin gestión de tenants ni de usuarios. |

- **Aislamiento multi-tenant:** el filtrado por `Id_Tenant` se aplica en la capa de servicio (los SP `GetBy_Id_Tenant*` reciben el tenant del contexto JWT). No hay Row-Level Security a nivel SQL (**GAP** — el aislamiento depende íntegramente del código de aplicación).
- **Bloqueo de cuenta:** 5 intentos fallidos ⇒ bloqueo 15 min (`autenticacion_v1.0.md §9`), gestionado sobre `sys_Usuarios.Intentos_Fallidos` / `Fecha_Bloqueo`.
- Roles admitidos por el CHECK de `sys_Usuarios.Rol`: `admin`, `operador`.

## 3. Permisos sobre la base de datos (plano SQL)

| Principal | Tipo | Permisos observados | Recomendación (least-privilege) |
|---|---|---|---|
| Cuenta de servicio de la app | Login SQL único | Acceso completo a la BD `IAConnect` (la conexión usa cuentas con privilegios amplios). Ejecuta CRUD vía SP. | Otorgar solo `EXECUTE` sobre los 75 SP y, si aplica, `SELECT` mínimo; **evitar** `db_owner`/`sysadmin`. |
| DBA | Login administrativo | Administración del esquema y mantenimiento. | Cuenta nominal, MFA, auditoría de accesos. |

**Observación de seguridad (GAP):** en los entornos de ejemplo la app se conecta con cuentas de alto privilegio (p. ej. `sa` en el compose de desarrollo). Se recomienda una cuenta de servicio dedicada con permisos mínimos (solo `EXECUTE` de SP).

## 4. Obtención de la cadena de conexión (sin valores)

La conexión se resuelve por la configuración estándar de .NET con la clave **`ConnectionStrings:IAConnect`**. Fuentes por entorno:

| Entorno | Mecanismo | Notas |
|---|---|---|
| Base (`appsettings.json`) | Clave `ConnectionStrings:IAConnect` **vacía** | Placeholder; obliga a proveer el valor por otra fuente. |
| Desarrollo local | `appsettings.Development.json` | Contiene una cadena de conexión de ejemplo. **No debe versionarse con credenciales reales** (GAP: hoy está en el repo — riesgo). |
| Contenedor | Variable de entorno `ConnectionStrings__IAConnect` (doble guión bajo = anidamiento de config .NET) | En `docker-compose.yml` se compone con la variable `${SA_PASSWORD}`; el password se inyecta desde el entorno, no está hardcodeado en el YAML. |
| Producción (recomendado) | Gestor de secretos (variables de entorno del orquestador / Azure Key Vault / User-Secrets en dev) | Nunca en control de versiones. |

Otros secretos relacionados, inyectados por variables de entorno (sin valores en este doc):

- **`IACONNECT_ENCRYPTION_KEY`** (variable de entorno): clave AES para **cifrar/descifrar `lut_Tenants.ApiKey_IA`** at-rest (verificado en `TenantService`/`Infrastructure`). Rotación ⇒ re-cifrado de las API keys. ⚠ Discrepancia reportada: `appsettings.json` declara `Encryption:AesKey` (no usado) y `docker-compose.yml` pasa `Encryption__Key`; el código lee la env var `IACONNECT_ENCRYPTION_KEY`. Si la clave falta, hay **fallback a texto plano** (`GAP-ENC-FALLBACK`). Ver [06-crosscutting](../01-architecture/06-crosscutting.md).
- **`Jwt:SecretKey`** (`Jwt__SecretKey`): firma de los JWT (access/refresh). Su compromiso invalida toda la seguridad de sesión.

> ⚠ El encabezado de `scripts/01_create_database.sql` y `appsettings.Development.json` incluyen un servidor/usuario/contraseña de ejemplo. **No se reproducen aquí** y deben eliminarse/rotarse antes de cualquier uso real.

## 5. Cifrado y protección de datos

- **En reposo:** `ApiKey_IA` cifrada a nivel aplicación (AES, clave por env `IACONNECT_ENCRYPTION_KEY`; ⚠ fallback a texto plano si falta); `Password_Hash` como hash bcrypt (`$2a$12$…`), no reversible. `Token` (refresh) y `Vector_Embedding` deben tratarse como secretos.
- **En tránsito:** las cadenas de ejemplo usan `TrustServerCertificate=true` (**GAP**: aceptable en dev, inseguro en prod; usar certificado válido).
- **Datos personales:** ver clasificación PII/secreto en [`data-dictionary.md` §4](./data-dictionary.md#4-columnas-pii--secreto-resumen).

## 6. Retención de datos (inferida — GAP)

El esquema **no define políticas de retención ni jobs de purga**, y las FKs son `NO ACTION` (sin cascada). Retención inferida a validar con negocio/legal:

| Tabla | Retención inferida | Riesgo / acción |
|---|---|---|
| lut_Tenants | Permanente (maestro); baja lógica `Activo` | Borrado físico bloqueado por FKs si hay dependientes. |
| sys_Usuarios | Permanente; baja lógica `Activo` | Ídem (referida por refresh tokens). |
| sys_Sesiones / sys_Mensajes | Sujeta a política de datos personales | **GAP**: sin purga; `Contenido` es PII → definir plazo y borrado. |
| sys_Fragmentos_Conocimiento | Ligada al documento origen | **GAP**: sin borrado en cascada por `Documento_Origen`. |
| sys_Metricas_Uso | Archivable por antigüedad | **GAP**: crecimiento append-only sin agregación/archivado. |
| sys_Refresh_Tokens | Purgable tras expiración/revocación | **GAP**: sin job de limpieza; acumula tokens muertos. |

## Referencias cruzadas

- Diccionario de datos: [`data-dictionary.md`](./data-dictionary.md)
- Modelo ER (código): [`er-diagrams/iaconnect.dbml`](./er-diagrams/iaconnect.dbml)
- Matriz de datos de prueba: [`test-data-matrix.md`](./test-data-matrix.md)
