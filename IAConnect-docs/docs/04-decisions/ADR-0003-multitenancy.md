---
doc_id: ADR-0003
doc_type: adr
title: "ADR-0003: Multi-tenancy por Id_Tenant con configuración por tenant"
status: draft
origin: reverse-engineered
confidence: high
date: 2026-07-15
deciders: [pendiente-asignacion]
owner: pendiente-asignacion
last_review: 2026-07-15
audience: [arquitectos, dev, seguridad, agentes-automaticos]
classification: uso-interno
traces: [ADR-0004]
supersedes: null
---

# ADR-0003: Multi-tenancy por Id_Tenant con configuración por tenant

> Decisión reconstruida. Confianza: alta (`TenantResolverMiddleware`, `TenantAccessFilter`, `lut_Tenants`).

## Contexto

Varias organizaciones deben usar la plataforma de forma aislada, cada una con su proveedor IA, modelo, *system
prompt*, límites y credenciales.

## Decisión

Modelar el tenant como entidad de primera clase (`lut_Tenants`, PK `Id_Tenant` de negocio). El `tenantId` viaja en
la **ruta**; `TenantResolverMiddleware` lo resuelve y rechaza tenants inexistentes/inactivos (404);
`TenantAccessFilter` autoriza (admin: cualquier tenant; operador: el propio). Todas las tablas de negocio
particionan por `Id_Tenant`. La configuración de IA vive por tenant.

## Alternativas

- **Base de datos por tenant:** mayor aislamiento físico, mayor costo operativo; no adoptado.
- **Tenant en el token únicamente (sin en ruta):** menos explícito para APIs multi-tenant administradas por admins.

## Consecuencias

- (+) Aislamiento lógico simple y explícito; configuración flexible por tenant.
- (+) Un admin puede operar sobre cualquier tenant; el operador queda acotado.
- (–) El aislamiento es **lógico** (misma BD): depende de que cada consulta filtre por `Id_Tenant`.
- (–) La API key del proveedor por tenant se guarda en la BD (cifrada; ver ADR-0004 y crosscutting).
