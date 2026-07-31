---
doc_id: DOCS-INDEX-001
doc_type: master-index
title: IAConnect-docs — Índice maestro
version: 1.0.0
status: draft
origin: reverse-engineered
confidence: high
owner: pendiente-asignacion
last_review: 2026-07-15
review_cycle_days: 180
audience: [todos, agentes-automaticos]
classification: uso-interno
traces: []
supersedes: null
---

# IAConnect-docs — Índice maestro

> **Resumen ejecutivo.** IAConnect es un **gateway multi-tenant de IA conversacional** (.NET 8, Clean Architecture):
> una API REST que enruta chat/completion/analyze/summarize/improve al proveedor IA configurado por cada tenant
> (Gemini/Claude/OpenAI), con autenticación JWT, memoria de conversación, base de conocimiento por tenant (RAG) y
> métricas de uso. Este conjunto documental fue **reconstruido por ingeniería inversa** desde el código; todo está
> en `status: draft` para revisión humana. El origen (`/NG/Ng-IAServices`) **no fue modificado**.

## Cómo navegar (humanos)

| Querés… | Ir a |
|---|---|
| Entender el propósito y alcance | [docs/00-overview/vision.md](docs/00-overview/vision.md) |
| Ver el inventario de piezas | [docs/00-overview/system-map.md](docs/00-overview/system-map.md) |
| Entender la arquitectura (C4, flujos) | [docs/01-architecture/](docs/01-architecture/01-context.md) |
| Requisitos y trazabilidad | [docs/02-requirements/](docs/02-requirements/traceability-matrix.md) |
| Modelo de datos (diccionario, ER, casos QA) | [docs/03-data/](docs/03-data/data-dictionary.md) |
| Decisiones de arquitectura (ADRs) | [docs/04-decisions/](docs/04-decisions/ADR-0001-clean-architecture.md) |
| Trabajo a desarrollar / re-evaluar (backlog técnico) | [docs/10-evolution/backlog-tecnico.md](docs/10-evolution/backlog-tecnico.md) |
| Contrato de la API (OpenAPI) | [docs/05-apis/](docs/05-apis/catalog.md) |
| Operar el servicio | [docs/07-operations/runbook-api.md](docs/07-operations/runbook-api.md) |
| Levantar el entorno | [docs/08-onboarding/developer-setup.md](docs/08-onboarding/developer-setup.md) |
| Documentación por pieza | [docs/pieces/](docs/pieces/) |
| Vocabulario | [GLOSSARY.md](GLOSSARY.md) |

## Cómo navegar (agentes)

- Estado documental máquina-legible: [`docs-manifest.yaml`](docs-manifest.yaml) (piezas, `type`, `required_docs`, gap).
- Contexto rápido sin releer código: [`../ia-db/README.md`](../ia-db/README.md).
- Política de agentes: [docs/09-agents/agent-policy.md](docs/09-agents/agent-policy.md).

## Mapa del conjunto documental

```text
IAConnect-docs/
├── README.md                  ← este índice maestro
├── GLOSSARY.md                ← vocabulario controlado
├── docs-manifest.yaml         ← manifiesto máquina-legible + gap
└── docs/
    ├── 00-overview/           vision · system-map
    ├── 01-architecture/       01-context · 02-containers · 03-components · 04-runtime-views · 05-deployment · 06-crosscutting
    ├── 02-requirements/       functional-overview · non-functional · traceability-matrix
    ├── 03-data/               data-dictionary · er-diagrams/iaconnect.dbml · access-policies · test-data-matrix · fixtures/
    ├── 04-decisions/          ADR-0001 … ADR-0006
    ├── 05-apis/               catalog · openapi.yaml
    ├── 07-operations/         runbook-api
    ├── 08-onboarding/         developer-setup
    ├── 09-agents/             agent-policy
    ├── 10-evolution/          backlog-tecnico  ← capacidades a desarrollar / re-evaluar
    └── pieces/                API · Application · Domain · Infrastructure · ChatWidget · Demo.Web · Example · Tests
```

> **Omisiones declaradas (arc42).** No se crearon `06-security-compliance/` (perfil regulado A.17 no aplica,
> `regulated=false`) ni carpetas sin contenido reconstruible. La seguridad transversal vive en
> [06-crosscutting](docs/01-architecture/06-crosscutting.md).

## Inventario de piezas

| Pieza | `type` | Doc |
|---|---|---|
| IAConnect.API | service | [README](docs/pieces/IAConnect.API/README.md) |
| IAConnect.Application | library | [README](docs/pieces/IAConnect.Application/README.md) |
| IAConnect.Domain | library | [README](docs/pieces/IAConnect.Domain/README.md) |
| IAConnect.Infrastructure | library | [README](docs/pieces/IAConnect.Infrastructure/README.md) |
| IAConnect.ChatWidget | component-library | [README](docs/pieces/IAConnect.ChatWidget/README.md) · [catálogo](docs/pieces/IAConnect.ChatWidget/component-catalog.md) |
| IAConnect.Demo.Web | web-portal | [README](docs/pieces/IAConnect.Demo.Web/README.md) · [rutas](docs/pieces/IAConnect.Demo.Web/routes-map.md) |
| IAConnect.Example | library (sample) | [README](docs/pieces/IAConnect.Example/README.md) |
| IAConnect.Tests | test-support | [README](docs/pieces/IAConnect.Tests/README.md) |
| Base de datos IAConnect | database | [03-data](docs/03-data/data-dictionary.md) |

## Estado documental (gap)

Calculable desde [`docs-manifest.yaml`](docs-manifest.yaml). Resumen:

- **Cobertura:** las 9 piezas tienen su documentación base; la API tiene OpenAPI inferido y runbook; la BD tiene
  diccionario + ER dbml + políticas de acceso + matriz de casos + fixtures sintéticos.
- **Gaps abiertos** (ver manifiesto): `GAP-DB-LIVE` (sin introspección en vivo), `GAP-CHANGELOG` (sin historial de
  versiones), `GAP-APIREF` (referencia de API generada), `GAP-A11Y` (accesibilidad), `GAP-TEST-COVERAGE`.

## Reporte de derivas (divergencias código ↔ diseño/doc previa)

Detectadas durante la documentación; **gana el código**, se reportan para decisión humana (Marco §6/§9/§12.4):

| ID | Deriva | Severidad | Dónde |
|---|---|---|---|
| `GAP-RAG-SEMANTIC` | RAG es **TF-IDF léxico**, no embeddings/coseno; `Vector_Embedding=null`; contradice `rag-spec_v1.0.md` | Alta | [ADR-0006](docs/04-decisions/ADR-0006-rag-por-tenant.md), [runtime-views](docs/01-architecture/04-runtime-views.md) |
| `GAP-ENC-FALLBACK` | API key de tenant cifrada AES (env `IACONNECT_ENCRYPTION_KEY`) con **fallback a texto plano**; nombres de config inconsistentes | Alta | [06-crosscutting](docs/01-architecture/06-crosscutting.md) |
| `GAP-CREDS-IN-REPO` | Credenciales de ejemplo versionadas (appsettings.Development, encabezado SQL, compose `sa`) | Alta | [access-policies](docs/03-data/access-policies.md), [deployment](docs/01-architecture/05-deployment.md) |
| `GAP-LAYER-INVERSION` | `Infrastructure.csproj` referencia a `Application` (Infra→App) | Media | [ADR-0001](docs/04-decisions/ADR-0001-clean-architecture.md) |
| `GAP-DB-INTEGRITY` | Integridad solo en app (sin CHECK en `Proveedor*`, sin RLS, sin retención) | Media | [data-dictionary](docs/03-data/data-dictionary.md) |
| `GAP-CONFIG-MODEL` | `AIProviders:*:DefaultModel` de config **no se consume** (modelo sale del tenant) | Baja | [ADR-0004](docs/04-decisions/ADR-0004-abstraccion-proveedor-ia.md) |
| — (menor) | Entidades usan `string` en `Proveedor_IA`/`Rol` en vez de los enums; `AuthService.GetUsuariosAsync` con tenant vacío; unicidad de usuario global (no por tenant); doble identidad de sesión (`Id` int vs `Id_Sesion` GUID) | Baja | pieces [Domain](docs/pieces/IAConnect.Domain/README.md), [Application](docs/pieces/IAConnect.Application/README.md), [Infrastructure](docs/pieces/IAConnect.Infrastructure/README.md) |

## Seguridad

Ningún entregable contiene credenciales, API keys ni datos productivos. Los ejemplos son **sintéticos**. Las derivas
`GAP-CREDS-IN-REPO` y `GAP-ENC-FALLBACK` señalan riesgos del **origen** que requieren remediación (rotación/limpieza),
fuera del alcance de esta documentación (que no modifica el origen).

## Próximos pasos (revisión humana)

1. Validar requisitos y criticidades con negocio; asignar `owner` a cada documento (hoy `pendiente-asignacion`).
2. Decidir sobre `GAP-RAG-SEMANTIC` (implementar embeddings vs. actualizar el diseño) y `GAP-ENC-FALLBACK` (hardening).
3. Remediar `GAP-CREDS-IN-REPO` en el origen.
4. Promover documentos revisados de `draft` → `approved`.
5. Mover cada `pieces/<pieza>/` al repositorio de su pieza cuando se apruebe (adaptación declarada del Profile).
