---
doc_id: ADR-0002
doc_type: adr
title: "ADR-0002: Persistencia con patrón DataEntity-DataManager sobre stored procedures"
status: draft
origin: reverse-engineered
confidence: high
date: 2026-07-15
deciders: [pendiente-asignacion]
owner: pendiente-asignacion
last_review: 2026-07-15
audience: [arquitectos, dev, dba, agentes-automaticos]
classification: uso-interno
traces: [ADR-0001]
supersedes: null
---

# ADR-0002: Persistencia con patrón DataEntity-DataManager sobre stored procedures

> Decisión reconstruida. Confianza: alta (evidenciada por `Infrastructure/DataAccess/DataEntityCore.cs`,
> `Infrastructure/DataManagers/**` y `scripts/01_create_database.sql`).

## Contexto

Se necesita acceso a datos sobre SQL Server con control fino de las consultas y alineado a un estándar corporativo
existente (hay un `PackageReference` comentado a un `DataEntityCore` corporativo).

## Decisión

Adoptar el patrón propietario **DataEntity-DataManager**: cada tabla expone stored procedures con la convención
`SP_<tabla>_<Add|Update|Delete|GetAll|GetOne|GetBy_<col>[_Cantidad]>`; un `*DataManager` por tabla los invoca a
través de `DataEntityCore` (conexión estática configurada al arranque, derivación de parámetros con
`SqlCommandBuilder.DeriveParameters`, asignación posicional y mapeo por reflexión a los `*Model`). **No se usa un
ORM (EF Core).**

## Alternativas

- **Entity Framework Core:** productividad y migraciones, pero menor control del SQL y desalineado al estándar corporativo.
- **Dapper:** micro-ORM ligero; no adoptado.

## Consecuencias

- (+) Control total del SQL; alineación al estándar corporativo; sin dependencia de ORM.
- (–) **Acoplamiento fuerte a SQL Server y a los SP**; el mapeo **posicional** de parámetros es frágil ante cambios
  de firma del SP; `DeriveParameters` agrega un round-trip por llamada (costo de rendimiento).
- (–) Los cambios de esquema exigen tocar SP + DataManager + Model de forma coordinada (ver §14 / diccionario de datos).
