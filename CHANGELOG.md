# Changelog

Todos los cambios relevantes de este repositorio documental se registran en este archivo.

El formato sigue [Keep a Changelog](https://keepachangelog.com/es-ES/1.1.0/) y el versionado
[SemVer](https://semver.org/lang/es/) aplicado al **conjunto documental** (no al código de origen).

> **Alcance.** Este repositorio contiene únicamente documentación de la solución **IAConnect**
> (`/NG/Ng-IAServices`). El código de origen **no se modifica** desde acá.

## [Unreleased]

### Pendiente

- Revisión humana y promoción de `status: draft` → `approved` en el conjunto `IAConnect-docs/`.
- Asignar `owner` en los front-matter (hoy `pendiente-asignacion`).
- Completar `Analisis/README.md` (actualmente vacío).
- Corregir el enlace a `ia-db/README.md` en el `README.md` raíz (hoy apunta a una ruta absoluta del workspace).

## [1.0.0] - 2026-07-31

Primera publicación del conjunto documental completo. Todo el contenido fue reconstruido por
**ingeniería inversa** desde el código fuente, conforme al
[Marco de Documentación de Soluciones de Software](../../IA.Prompting.Templates/Referencias/Marco-Documentacion-Software-v1.md)
(Tool-Prompt `Documentar-Fuentes-Software.md`, profile `Solution-Documentation`).

### Añadido

**`IAConnect-docs/` — conjunto documental (Marco §5.2)**

- Índice maestro (`README.md`), vocabulario controlado (`GLOSSARY.md`) y manifiesto máquina-legible
  con cálculo de gap (`docs-manifest.yaml`).
- `00-overview/`: `vision.md`, `system-map.md`.
- `01-architecture/`: contexto, contenedores, componentes, vistas de runtime, despliegue y
  aspectos transversales (C4 + flujos).
- `02-requirements/`: panorama funcional, requisitos no funcionales y matriz de trazabilidad.
- `03-data/`: diccionario de datos, diagrama ER (`iaconnect.dbml`), políticas de acceso,
  matriz de datos de prueba y fixtures sintéticos (`iaconnect.seed.yaml`).
- `04-decisions/`: ADR-0001 Clean Architecture, ADR-0002 DataEntity/DataManager + SP,
  ADR-0003 multi-tenancy, ADR-0004 abstracción de proveedor IA, ADR-0005 JWT + refresh tokens,
  ADR-0006 RAG por tenant.
- `05-apis/`: catálogo de endpoints y contrato `openapi.yaml`.
- `07-operations/`: `runbook-api.md`.
- `08-onboarding/`: `developer-setup.md`.
- `09-agents/`: `agent-policy.md`.
- `10-evolution/`: `backlog-tecnico.md`.
- `pieces/`: documentación por proyecto — API, Application, Domain, Infrastructure, ChatWidget
  (con catálogo de componentes), Demo.Web (con mapa de rutas), Example y Tests.

**`ia-db/` — base de conocimiento indexada (Knowledge-Indexing)**

- Punto de entrada (`README.md`) con tabla de navegación para agentes IA.
- Índices: `00_MASTER-INDEX`, `01_arquitectura`, `02_dominio-y-datos`, `03_api-endpoints`,
  `04_proveedores-ia-y-rag`, `05_seguridad-y-multitenant`, `06_pruebas-y-devops`.

**`Analisis/` — estudio de integración de asistencia por IA**

- `Implementacion/Analisis/`: planteo de contexto, INPUTs (concepto y usuarios de turnos) y
  OUTPUTs (marco de referencia, mapa conceptual, base de conocimiento y diagnóstico, estructura y
  plantilla KB, metodologías y catalogación, integración IAConnect, flujos conversacionales,
  información dinámica, glosario) más anexos A1 (plantilla KB) y A2 (checklist de evaluación).
- `Implementacion/Antecedentes/`: análisis de asistencia IA en ChatBotIA, relevamiento y
  verificación de BoleteriaCore, referencia IA de Mercado Libre y material gráfico.
- `Implementacion/Ng-IAServices/`: SAD, HLD, LLD, ADR, Operations Guide y Administrator Guide del
  bloque metodológico común.
- `Implementacion/GDA-Turnos/` y `Implementacion/Boleteria-Eventos/`: casos de éxito con SAD, HLD,
  LLD, ADR, Operations Guide, Administrator Guide y plan de sprints/capacitación.

**`PROMPTs/`**

- Tool-prompts usados para generar el conjunto: estudio sobre asistencia al usuario por IA,
  documentación de `Ng-IAServices`, análisis de `Ng-IAServices` e integración GDA/Boletería.

### Cambiado

- `README.md` raíz: pasó de placeholder a portada del repositorio, con guía de navegación,
  estructura del árbol documental y advertencia de seguridad.

### Seguridad

- Ningún entregable incluye credenciales, cadenas de conexión con secretos ni datos personales
  reales; todos los ejemplos usan datos **sintéticos**.

[Unreleased]: https://github.com/Notions-Group-SA/Ng-IAServices.Documentacion/compare/v1.0.0...HEAD
[1.0.0]: https://github.com/Notions-Group-SA/Ng-IAServices.Documentacion/releases/tag/v1.0.0
