---
doc_id: ADR-0001
doc_type: adr
title: "ADR-0001: Clean Architecture en capas"
status: draft
origin: reverse-engineered
confidence: high
date: 2026-07-15
deciders: [pendiente-asignacion]
owner: pendiente-asignacion
last_review: 2026-07-15
audience: [arquitectos, dev, agentes-automaticos]
classification: uso-interno
traces: []
supersedes: null
---

# ADR-0001: Clean Architecture en capas

> Decisión **reconstruida** por ingeniería inversa (no existe ADR original). Confianza: alta (evidenciada por la
> estructura de proyectos y las referencias de `.csproj`).

## Contexto

La solución debe integrar múltiples proveedores IA y una persistencia propia manteniendo la lógica de negocio
independiente de detalles de infraestructura (BD, HTTP a proveedores, framework web).

## Decisión

Organizar el código en cuatro capas con la regla de dependencia apuntando al centro:
`IAConnect.Domain` (núcleo) ← `IAConnect.Application` ← `IAConnect.Infrastructure` / `IAConnect.API`. El dominio no
tiene dependencias externas (verificado: `IAConnect.Domain.csproj` sin `PackageReference` ni `ProjectReference`).

## Alternativas

- **Arquitectura por capas tradicional (N-tier) con acoplamiento a EF/BD:** más simple pero acopla negocio a datos.
- **Vertical slices / modular monolith:** válido, pero el equipo optó por capas clásicas de Clean Architecture.

## Consecuencias

- (+) Negocio testeable en aislamiento; proveedores y BD intercambiables detrás de interfaces del dominio.
- (+) Contratos (`IAIProvider`, `I*DataManager`) definidos en `Domain`, implementados en `Infrastructure`.
- (–) **Desviación detectada:** `IAConnect.Infrastructure.csproj` referencia a `IAConnect.Application`
  (Infra→App), lo que **invierte** la regla de dependencia en ese borde. Registrado para corrección.
