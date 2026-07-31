---
doc_id: OVW-VIS-001
doc_type: vision
title: Visión y alcance — IAConnect
version: 1.0.0
status: draft
origin: reverse-engineered
confidence: high
owner: pendiente-asignacion
last_review: 2026-07-15
review_cycle_days: 180
audience: [stakeholders, arquitectos, dev, agentes-automaticos]
classification: uso-interno
traces: []
supersedes: null
---

# Visión y alcance — IAConnect

> **Resumen ejecutivo.** IAConnect es un *gateway* multi-tenant que unifica el acceso a varios proveedores de
> IA conversacional (Google Gemini, Anthropic Claude, OpenAI) tras una única API REST. Cada organización (tenant)
> configura su proveedor, modelo, *system prompt* y límites; la solución enruta las solicitudes, mantiene la
> memoria de conversación, ofrece una base de conocimiento por tenant (RAG) y registra métricas de uso.

## Problema que resuelve

Integrar IA conversacional en productos exige lidiar con múltiples proveedores (APIs distintas, autenticación,
modelos), aislamiento entre clientes, memoria de conversación, gobierno de costos y una capa de conocimiento
propia. IAConnect encapsula todo eso: **el consumidor habla con una sola API**; el proveedor concreto es un
detalle de configuración del tenant.

## Propuesta de valor

- **Independencia de proveedor:** cambiar de Gemini a Claude u OpenAI es configuración (`lut_Tenants.Proveedor_IA`),
  no código.
- **Multi-tenant de raíz:** datos, configuración y credenciales aislados por tenant.
- **Capacidades de texto llave en mano:** chat, completion, analyze, summarize, improve.
- **RAG por tenant:** carga de documentos y recuperación de contexto.
- **Observabilidad de consumo:** métricas de tokens/latencia por tenant y proveedor.

## Alcance

| En alcance | Fuera de alcance (hoy) |
|---|---|
| API REST multi-tenant, auth JWT + refresh | Facturación / *billing* automatizado |
| Proveedores Gemini/Claude/OpenAI | *Streaming* de respuestas (verificar en providers) |
| RAG con fragmentos + embeddings | Búsqueda vectorial avanzada / re-ranking (verificar) |
| Widget Blazor + web demo | App móvil |
| Métricas de uso | Panel analítico dedicado |

## Stakeholders (inferidos)

| Rol | Interés |
|---|---|
| Administrador de plataforma | Alta/baja de tenants y usuarios, configuración de proveedores |
| Operador de tenant | Uso de las capacidades IA dentro de su tenant |
| Integrador/Desarrollador | Consumir la API o embeber el widget |
| Usuario final | Conversar con el asistente (vía widget/portal del integrador) |

> Stakeholders reconstruidos desde el código y `docs/01_contexto/stakeholders_v1.0.md` del origen; validar con negocio.

## Casos de uso principales (origen: `docs/02_especificacion_funcional/casos-de-uso/`)

CU-01 Chat multi-turno · CU-02 Completion · CU-03 Analyze · CU-04 Summarize · CU-05 Improve ·
CU-06 Gestionar tenant · CU-07 Cargar conocimiento (RAG). Trazabilidad → [requisitos](../02-requirements/traceability-matrix.md).
