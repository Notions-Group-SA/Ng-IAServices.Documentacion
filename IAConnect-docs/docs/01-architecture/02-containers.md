---
doc_id: ARCH-CNT-001
doc_type: architecture-containers
title: Contenedores (C4-L2) — IAConnect
version: 1.0.0
status: draft
origin: reverse-engineered
confidence: high
owner: pendiente-asignacion
last_review: 2026-07-15
review_cycle_days: 180
audience: [arquitectos, dev, ops, agentes-automaticos]
classification: uso-interno
traces: [ADR-0001, ADR-0002]
supersedes: null
---

# Contenedores (C4 — Nivel 2) — IAConnect

> **Resumen ejecutivo.** La solución se despliega como una **API ASP.NET Core** (única unidad ejecutable de
> servidor) respaldada por **SQL Server**, más artefactos cliente (widget Blazor RCL, web demo Blazor Server,
> ejemplo de consola). La API concentra la lógica; los clientes la consumen por HTTP+JWT.

## Diagrama de contenedores

```mermaid
flowchart TB
    subgraph Clientes
        Web[Demo.Web\nBlazor Server]
        Widget[ChatWidget\nRazor Class Library]
        Example[Example\nConsola]
    end
    subgraph Servidor
        API[IAConnect.API\nASP.NET Core Web API :8080]
    end
    DB[(SQL Server 2022\nIAConnect)]
    P1[[Gemini]]
    P2[[Claude]]
    P3[[OpenAI]]

    Web --> Widget
    Widget -->|HTTP JWT| API
    Web -->|HTTP JWT| API
    Example -->|HTTP JWT| API
    API -->|DataManagers / SP| DB
    API -->|HttpClient| P1
    API -->|HttpClient| P2
    API -->|HttpClient| P3
```

## Contenedores y responsabilidades

| Contenedor | Ejecutable | Tecnología | Responsabilidad |
|---|---|---|---|
| IAConnect.API | Sí (`dotnet IAConnect.API.dll`) | ASP.NET Core Web API | Autenticación, autorización, resolución de tenant, orquestación IA, RAG, métricas, Swagger |
| SQL Server `IAConnect` | Sí (contenedor `mssql/server:2022`) | SQL Server | Persistencia (7 tablas + SP CRUD) |
| IAConnect.Demo.Web | Sí | Blazor Server | Portal de demostración de las capacidades |
| IAConnect.ChatWidget | No (librería) | Blazor RCL | Componente de chat embebible |
| IAConnect.Example | Sí (consola) | .NET 8 | Ejemplo de integración |

## Módulos internos de la API (referencias de proyecto)

`IAConnect.API` compone en tiempo de ejecución (DI en `Program.cs`) tres librerías:

| Librería | Aporta |
|---|---|
| IAConnect.Domain | Entidades, enums, excepciones, contratos (interfaces) |
| IAConnect.Application | Servicios de casos de uso, DTOs |
| IAConnect.Infrastructure | DataManagers (SP), `DataEntityCore`, proveedores IA + factory |

Detalle de componentes internos → [03-components.md](03-components.md).

## Comunicación

| Origen → Destino | Protocolo | Auth |
|---|---|---|
| Clientes → API | HTTP/HTTPS REST | JWT Bearer |
| API → SQL Server | TDS (ADO.NET / SP) | Cadena `ConnectionStrings:IAConnect` |
| API → Proveedores IA | HTTPS | API key por tenant (`lut_Tenants.ApiKey_IA`) / config |

## Puertos (observados)

- API: `8080` (contenedor), `5051` http / `7167` https (dev, `launchSettings.json`).
- SQL Server: `1433` (docker-compose).

Despliegue e infraestructura → [05-deployment.md](05-deployment.md).
