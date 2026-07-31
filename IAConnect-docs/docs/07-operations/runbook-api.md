---
doc_id: OPS-RB-001
doc_type: runbook
title: Runbook — IAConnect.API
version: 1.0.0
status: draft
origin: reverse-engineered
confidence: medium
owner: pendiente-asignacion
last_review: 2026-07-15
review_cycle_days: 180
audience: [ops, dev, soporte, agentes-automaticos]
classification: uso-interno
traces: []
supersedes: null
---

# Runbook — IAConnect.API

> **Resumen ejecutivo.** Operación básica del servicio: arranque, salud, diagnóstico de fallos frecuentes y
> escalamiento. Reconstruido; validar ejecutándolo (game day) antes de promover a `approved`.

## Servicio

- Ejecutable: `dotnet IAConnect.API.dll` (contenedor `iaconnect-api`, puerto 8080).
- Dependencias: SQL Server `IAConnect` (1433) y conectividad HTTPS a los proveedores IA.

## Salud

| Check | Cómo | Sano |
|---|---|---|
| Liveness/Readiness | `GET /health` | 200 |
| Raíz | `GET /` | `{ Status: "Running", Service: "IAConnect API", Version }` |
| SQL Server | healthcheck compose (`SELECT 1`) | healthy |

## Detección → diagnóstico → mitigación

| Síntoma | Causa probable | Diagnóstico | Mitigación |
|---|---|---|---|
| 500 al arrancar | `ConnectionStrings:IAConnect` ausente/incorrecta | logs de arranque; `DataEntityCore.Configure` | Cargar cadena válida (gestor de secretos) |
| 401 en todo | `Jwt:SecretKey` distinto al que firmó el token, o expirado | claims/exp del token | Corregir secreto/issuer/audience; re-login |
| 404 "Tenant no encontrado o inactivo" | tenant inexistente o `Activo=0` | `SP_lut_Tenants_GetOne` | Activar/crear tenant |
| 403 "No tiene acceso a este tenant" | operador accediendo a otro tenant | claim `id_tenant` vs ruta | Usar tenant propio o rol admin |
| 502 en endpoints IA | proveedor caído/mal configurado (`ProviderUnavailableException`) | logs; API key/modelo del tenant | Verificar API key/modelo; reintento (Claude reintenta solo) |
| 423 en login | cuenta bloqueada por 5 intentos | `sys_Usuarios.Fecha_Bloqueo` | Esperar 15 min o desbloquear (admin/DBA) |
| Respuestas RAG pobres | RAG léxico (TF-IDF), no semántico | ver ADR-0006 | Cargar fragmentos con vocabulario alineado; roadmap: embeddings |

## Base de datos

- Crear/actualizar esquema: `sqlcmd -S <server> -U <user> -P <secreto-del-gestor> -i scripts/01_create_database.sql`.
  **Nunca** versionar/loguear la cadena con credenciales.
- Backup/retención: **no documentado en el origen** → definir (gap operativo). Ver [access-policies](../03-data/access-policies.md).

## Escalamiento

1. Revisar logs (5xx se loguean como `Error`).
2. Aislar: ¿falla BD, proveedor IA, o la app? (health + endpoint raíz + prueba directa al proveedor).
3. Escalar a: owner de la API (pendiente asignación) → DBA (si BD) → proveedor IA (si el fallo es externo).

## Gaps operativos

- Sin tracing/correlación; sin métricas de infraestructura exportadas; sin panel de las `sys_Metricas_Uso`.
- Sin política de backup/restore documentada; sin manifiestos productivos (k8s).
