> **Punto de entrada único de la ia-db de IAConnect.** Este archivo es la puerta de acceso a la base de
> conocimiento. Un agente de IA que necesite contexto de la solución debe: (1) leer la tabla de navegación,
> (2) cargar **solo** el/los índices relevantes de `indexes/`, (3) ampliar al código fuente solo ante
> insuficiencia comprobada. No recorrer el repositorio completo cuando existe información indexada.

# ia-db — IAConnect

Base de conocimiento destilada de la solución **IAConnect** (`/NG/Ng-IAServices`). Cada índice condensa un
dominio y referencia las fuentes para el detalle. Reduce el consumo de tokens y acelera el arranque de contexto.

## Tabla de navegación

| Necesitás saber… | Leé este índice |
|---|---|
| Visión general, stack, proyectos, decisiones clave | [`00_MASTER-INDEX.md`](indexes/00_MASTER-INDEX.md) |
| Cómo se estructura la solución (Clean Architecture, capas, dependencias) | [`01_arquitectura.md`](indexes/01_arquitectura.md) |
| Modelo de dominio, entidades, enums y modelo de datos (7 tablas) | [`02_dominio-y-datos.md`](indexes/02_dominio-y-datos.md) |
| Endpoints REST, controladores, DTOs, contrato de la API | [`03_api-endpoints.md`](indexes/03_api-endpoints.md) |
| Proveedores IA (Gemini/Claude/OpenAI), factory, RAG, prompts | [`04_proveedores-ia-y-rag.md`](indexes/04_proveedores-ia-y-rag.md) |
| Seguridad: JWT, multi-tenant, aislamiento, autorización | [`05_seguridad-y-multitenant.md`](indexes/05_seguridad-y-multitenant.md) |
| Pruebas, DevOps, despliegue, configuración | [`06_pruebas-y-devops.md`](indexes/06_pruebas-y-devops.md) |

## Resumen ejecutivo (1 pantalla)

- **Solución:** IAConnect — pasarela (gateway) multi-tenant de servicios de IA conversacional.
- **Tipo:** solución multi-proyecto (.NET 8). REST API + librerías de capa + widget Blazor + web demo + base de datos.
- **Stack:** C# / .NET 8, ASP.NET Core Web API, Blazor (Server + Razor Class Library), SQL Server, JWT.
- **Repo origen:** `/NG/Ng-IAServices` (git, rama `main`).
- **Función principal:** exponer una API REST por tenant que enruta chat/completion/analyze/summarize/improve
  al proveedor IA configurado (Gemini, Claude u OpenAI), con memoria de conversación, base de conocimiento (RAG)
  y métricas de uso, todo aislado por tenant.
- **Arquitectura en una línea:** Clean Architecture (Domain ← Application ← Infrastructure ← API) con patrón
  propietario **DataEntity-DataManager** sobre stored procedures de SQL Server, y una **factory de proveedores IA**
  seleccionada por la configuración del tenant.

## Estructura de la ia-db

```text
ia-db/
├── README.md                        ← este punto de entrada
└── indexes/
    ├── 00_MASTER-INDEX.md           ← visión, stack, proyectos, decisiones
    ├── 01_arquitectura.md           ← capas, dependencias, patrones, flujos
    ├── 02_dominio-y-datos.md        ← entidades, enums, 7 tablas, SP, FKs
    ├── 03_api-endpoints.md          ← controladores, endpoints, DTOs, códigos
    ├── 04_proveedores-ia-y-rag.md   ← factory IA, providers, RAG, embeddings
    ├── 05_seguridad-y-multitenant.md← JWT, refresh tokens, tenant isolation
    └── 06_pruebas-y-devops.md       ← tests, Docker, config, despliegue
```

## Restricciones para IA que consuma este índice

- **Solo lectura del origen.** No proponer cambios al código de `/NG/Ng-IAServices` desde esta base.
- **Sin secretos.** Ningún índice contiene API keys, contraseñas ni cadenas de conexión reales; si se necesitan,
  se obtienen del gestor de secretos/configuración del origen, nunca desde aquí.
- **Trazabilidad.** Toda afirmación de un índice referencia su fuente (`ruta` en el origen). Lo no verificado se marca.
- La documentación humana completa vive en `../IAConnect-docs/`; la ia-db es un punto de entrada, no una copia.

## Manifiesto de generación

- Generado por : `/IA.Prompting.Templates/Tool-Prompts/Documentar-Fuentes-Software.md` (Profile `Solution-Documentation`)
- Alcance      : solución IAConnect — 8 proyectos .NET + base de datos SQL Server IAConnect
- Fuentes      : `/NG/Ng-IAServices/**` (código C#/Razor, `scripts/*.sql`, `docs/**`, `Dockerfile`, `docker-compose.yml`)
- Exclusiones  : `.git`, `bin`, `obj`, `packages`, binarios, la propia ia-db
- Generado     : 2026-07-15 · Versión: 1.0 · Origin: reverse-engineered · Status: draft
- Actualizar   : `/IA.Prompting.Templates/Tool-Prompts/Actualizar-Indexado.md` (o re-ejecutar el Tool-Prompt en modo incremental)
