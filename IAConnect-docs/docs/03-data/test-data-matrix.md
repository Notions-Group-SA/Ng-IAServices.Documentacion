---
doc_id: TEST-MATRIX-001
doc_type: test-data-matrix
title: "Matriz de datos de prueba — IAConnect"
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
  - NG/Ng-IAServices/docs/05_arquitectura_tecnica/autenticacion_v1.0.md
related:
  - ./data-dictionary.md
  - ./er-diagrams/iaconnect.dbml
  - ./fixtures/iaconnect.seed.yaml
---

# Matriz de datos de prueba — IAConnect

> Casos `TC-` derivados del **modelo de datos** (§15 del Marco): dominios/CHECK, unicidad, integridad referencial (FKs) y reglas de negocio. Cada caso es trazable a una restricción del esquema o a una regla documentada. Los datos usados son sintéticos (ver [`fixtures/iaconnect.seed.yaml`](./fixtures/iaconnect.seed.yaml)).

**Técnicas:** equivalencia (partición de clases válidas/inválidas) · borde (valores límite) · unicidad · integridad (FK/huérfanos) · dominio (CHECK/enum) · regla (negocio).

## 1. Dominio / CHECK constraints

| TC | Origen (modelo) | Técnica | Entrada | Esperado | Traza |
|---|---|---|---|---|---|
| TC-01 | `lut_Tenants.Proveedor_IA` CHECK | dominio (válido) | `Proveedor_IA = 'claude'` | INSERT OK | DDL:35 / RF-tenant |
| TC-02 | `lut_Tenants.Proveedor_IA` CHECK | dominio (válido) | `'gemini'`, `'openai'` | INSERT OK cada uno | DDL:35 |
| TC-03 | `lut_Tenants.Proveedor_IA` CHECK | dominio (inválido) | `Proveedor_IA = 'mistral'` | Falla CHECK (error 547) | DDL:35 |
| TC-04 | `lut_Tenants.Proveedor_IA` CHECK | borde (case) | `Proveedor_IA = 'Claude'` (mayúscula) | Falla CHECK (valores en minúscula) | DDL:35 |
| TC-05 | `sys_Usuarios.Rol` CHECK | dominio (válido) | `Rol = 'admin'` / `'operador'` | INSERT OK | DDL:67 |
| TC-06 | `sys_Usuarios.Rol` CHECK | dominio (inválido) | `Rol = 'supervisor'` | Falla CHECK | DDL:67 |
| TC-07 | `sys_Mensajes.Rol` CHECK | dominio (válido) | `Rol ∈ {user, assistant, system}` | INSERT OK | DDL:111 |
| TC-08 | `sys_Mensajes.Rol` CHECK | dominio (inválido) | `Rol = 'bot'` | Falla CHECK | DDL:111 |
| TC-09 | `sys_Metricas_Uso.Proveedor` (sin CHECK) | dominio (gap) | `Proveedor = 'proveedor-inexistente'` | **INSERT OK** (no hay CHECK) → validar en app | DDL:158 / **GAP** |
| TC-10 | `sys_Mensajes.Proveedor_Usado` (sin CHECK) | dominio (gap) | `Proveedor_Usado = 'xyz'` | **INSERT OK** (no hay CHECK) → validar en app | DDL:115 / **GAP** |
| TC-11 | `lut_Tenants.Temperatura` (sin CHECK) | borde | `Temperatura = 2.50` / `-1.00` | INSERT OK a nivel BD (rango 0–2 solo en app) | DDL:38 / **GAP** |
| TC-12 | `lut_Tenants.Temperatura` decimal(3,2) | borde (precisión) | `Temperatura = 0.7` (default) | Almacena `0.70` | DDL:38 |

## 2. Unicidad (UNIQUE)

| TC | Origen (modelo) | Técnica | Entrada | Esperado | Traza |
|---|---|---|---|---|---|
| TC-13 | `sys_Usuarios.Nombre_Usuario` UNIQUE | unicidad | Alta de 2 usuarios con mismo `Nombre_Usuario` | 2º INSERT falla (violación UNIQUE 2627) | DDL:63 |
| TC-14 | `sys_Usuarios.Nombre_Usuario` | equivalencia | `Nombre_Usuario` distinto en cada alta | Ambos INSERT OK | DDL:63 |
| TC-15 | `sys_Sesiones.Id_Sesion` UNIQUE (NEWID) | unicidad | Insertar sesión sin especificar `Id_Sesion` | GUID autogenerado, único | DDL:88 |
| TC-16 | `sys_Sesiones.Id_Sesion` UNIQUE | unicidad | Forzar mismo GUID en 2 sesiones | 2º INSERT falla (UNIQUE) | DDL:88 |
| TC-17 | `sys_Refresh_Tokens.Token` UNIQUE | unicidad | Insertar 2 refresh tokens con mismo `Token` | 2º INSERT falla (UNIQUE) | DDL:184 |

## 3. Integridad referencial (FKs / huérfanos)

| TC | Origen (modelo) | Técnica | Entrada | Esperado | Traza |
|---|---|---|---|---|---|
| TC-18 | FK `sys_Usuarios.Id_Tenant`→lut_Tenants | integridad | Usuario con `Id_Tenant = 'no-existe'` | Falla FK (547) | DDL:76 |
| TC-19 | FK `sys_Usuarios.Id_Tenant` nullable | equivalencia | Usuario `admin` con `Id_Tenant = NULL` | INSERT OK (admin global) | DDL:68,76 |
| TC-20 | FK `sys_Sesiones.Id_Tenant`→lut_Tenants | integridad | Sesión con tenant inexistente | Falla FK | DDL:98 |
| TC-21 | FK `sys_Mensajes.Id_Sesion`→sys_Sesiones.Id | integridad | Mensaje con `Id_Sesion` inexistente | Falla FK | DDL:123 |
| TC-22 | FK `sys_Mensajes.Id_Sesion` | integridad (referencia correcta) | Mensaje con `Id_Sesion` = `Id` (int) de sesión existente | INSERT OK (usa PK interna, no el GUID) | DDL:123 |
| TC-23 | FK `sys_Fragmentos_Conocimiento.Id_Tenant` | integridad | Fragmento con tenant inexistente | Falla FK | DDL:144 |
| TC-24 | FK `sys_Metricas_Uso.Id_Tenant` | integridad | Métrica con tenant inexistente | Falla FK | DDL:170 |
| TC-25 | FK `sys_Metricas_Uso.Id_Sesion` nullable | equivalencia | Métrica con `Id_Sesion = NULL` | INSERT OK (operación sin sesión) | DDL:157,171 |
| TC-26 | FK `sys_Refresh_Tokens.Id_Usuario` | integridad | Token con `Id_Usuario` inexistente | Falla FK | DDL:192 |
| TC-27 | FK `lut_Tenants` (NO ACTION) | integridad (borrado) | DELETE de tenant con usuarios/sesiones dependientes | Falla (sin cascada; error 547) | DDL:76,98,144,170 |
| TC-28 | FK `sys_Usuarios` (NO ACTION) | integridad (borrado) | DELETE de usuario con refresh tokens | Falla (sin cascada) | DDL:192 |

## 4. NOT NULL / defaults

| TC | Origen (modelo) | Técnica | Entrada | Esperado | Traza |
|---|---|---|---|---|---|
| TC-29 | `lut_Tenants.System_Prompt` NOT NULL | equivalencia (inválido) | INSERT sin `System_Prompt` | Falla NOT NULL (515) | DDL:36 |
| TC-30 | `lut_Tenants` defaults | equivalencia | INSERT mínimo (omitir Temperatura, Max_Tokens, Activo…) | Aplica defaults 0.7 / 4000 / 1 / 'PNG,JPG,WEBP' / 60 / 7 | DDL:38-46 |
| TC-31 | `sys_Usuarios.Password_Hash` NOT NULL | equivalencia (inválido) | INSERT sin `Password_Hash` | Falla NOT NULL | DDL:64 |
| TC-32 | `sys_Usuarios.Email` nullable | equivalencia | Usuario sin `Email` | INSERT OK (NULL) | DDL:66 |

## 5. Reglas de negocio

| TC | Origen (modelo) | Técnica | Entrada | Esperado | Traza |
|---|---|---|---|---|---|
| TC-33 | Bloqueo por intentos (`Intentos_Fallidos`/`Fecha_Bloqueo`) | regla (borde) | 5º login fallido consecutivo | `Intentos_Fallidos = 5`, `Fecha_Bloqueo = ahora+15min`; login devuelve 423 | autenticacion_v1.0 §9 / RNF-seguridad |
| TC-34 | Bloqueo por intentos | regla (borde inferior) | 4 fallidos y luego éxito | `Intentos_Fallidos` se resetea a 0; sin bloqueo | autenticacion_v1.0 §9 |
| TC-35 | Bloqueo temporal | regla | Login válido con `Fecha_Bloqueo` en el futuro | Rechazo 423 (cuenta bloqueada) aunque la contraseña sea correcta | autenticacion_v1.0 §9 |
| TC-36 | Refresh token revocado | regla | Refresh con `Revocado = 1` o `Fecha_Expiracion < ahora` | Rechazo; no emite nuevo access token | DDL:184-187 |
| TC-37 | Consistencia `Total_Tokens` | regla (sin trigger) | `Total_Tokens ≠ Tokens_Prompt + Tokens_Respuesta` | **INSERT OK** en BD (no validado) → validar en app | DDL:162 / **GAP** |
| TC-38 | Multi-tenant aislamiento | regla | `operador` de tenant A consulta sesiones de tenant B | La API debe filtrar por `Id_Tenant`; BD no lo impide (sin RLS) | multitenant-spec / **GAP** |

## 6. Gaps de cobertura

- **Sin CHECK a nivel BD** (validación solo en app): `sys_Metricas_Uso.Proveedor` (TC-09), `sys_Mensajes.Proveedor_Usado` (TC-10), rango de `Temperatura` (TC-11), rangos de `Access_Token_Expiracion_Minutos` (5–1440) y `Refresh_Token_Expiracion_Dias` (1–90), y `Total_Tokens` (TC-37).
- **Sin Row-Level Security:** el aislamiento multi-tenant (TC-38) depende del código; no hay prueba posible a nivel BD.
- **Sin cascada de borrado:** TC-27/TC-28 confirman `NO ACTION`; falta definir estrategia (soft-delete vs. cascada) y sus casos.
- **Columnas sin caso dedicado** (baja criticidad): `Indice_Fragmento` (orden), `Duracion_Ms`, `Tamano_Imagen_KB`, `Max_Tamano_Imagen_KB` (validación de tamaño de imagen: regla de app no trazable a constraint BD).
- **Formatos_Imagen_Permitidos:** formato CSV libre; sin validación de valores en BD (posible caso de dominio a nivel app).

## Referencias cruzadas

- Diccionario de datos: [`data-dictionary.md`](./data-dictionary.md)
- Modelo ER (código): [`er-diagrams/iaconnect.dbml`](./er-diagrams/iaconnect.dbml)
- Fixtures sintéticos: [`fixtures/iaconnect.seed.yaml`](./fixtures/iaconnect.seed.yaml)
