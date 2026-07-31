---
doc_id: DOCS-BACKLOG-001
doc_type: backlog
title: IAConnect-docs — Backlog técnico y evolución
version: 1.0.0
status: draft
origin: analysis
confidence: high
owner: pendiente-asignacion
last_review: 2026-07-15
review_cycle_days: 90
audience: [arquitectos, dev, producto, agentes-automaticos]
classification: uso-interno
traces: [ADR-0004, ADR-0006, GAP-RAG-SEMANTIC]
supersedes: null
---

# IAConnect-docs — Backlog técnico y evolución

> **Qué es este documento.** Registro de **capacidades a desarrollar / re-evaluar** en IAConnect, derivado del
> estudio [`../../../Analisis/Analisis-Asistencia-IA-ChatBotIA.md`](../../../Analisis/Analisis-Asistencia-IA-ChatBotIA.md)
> y de la revisión del código. **Son propuestas, no trabajo ejecutado:** el código de origen (`/NG/Ng-IAServices`)
> **no fue modificado** y ninguna de estas evoluciones está implementada. El objetivo es que, al consultar el SDD,
> quede visible **qué falta** y **qué hay que reevaluar** antes de decidir construirlo.

> **No confundir con el "Reporte de derivas" del índice maestro.** Aquel registra **divergencias código↔diseño
> previo** (`GAP-*`, "gana el código"). Este backlog registra **capacidades futuras** aún no diseñadas ni
> construidas. Cuando un ítem se relaciona con una deriva existente, se enlaza (ej.: `EVO-001` ↔ `GAP-RAG-SEMANTIC`).

## Convención

- **Estados:** `propuesto` → `a-re-evaluar` → `aceptado` → `implementado` | `descartado`.
- **Prioridad / Esfuerzo:** Baja / Media / Alta (cualitativos; a validar por negocio y arquitectura).
- Cada ítem declara **criterios de re-evaluación**: qué mirar para decidir avanzar, posponer o descartar.

## Resumen

| ID | Título | Estado | Prioridad | Esfuerzo | Relación |
|---|---|---|---|---|---|
| [EVO-001](#evo-001--recuperación-rag-semántica--híbrida) | Recuperación RAG semántica / híbrida | `a-re-evaluar` | Alta | Media | `GAP-RAG-SEMANTIC`, [ADR-0006](../04-decisions/ADR-0006-rag-por-tenant.md) |
| [EVO-002](#evo-002--function-calling--tools-para-datos-dinámicos-del-usuario) | Function-calling / tools para datos dinámicos | `propuesto` | Media-Alta | Alta | [ADR-0004](../04-decisions/ADR-0004-abstraccion-proveedor-ia.md) |

> Relación funcional (estudio §B2): **EVO-001** habilita el **conocimiento estático** con significado; **EVO-002**
> habilita los **datos/acciones dinámicos por usuario**. Son independientes y complementarios; un asistente maduro
> (referencia observada: Mercado Pago, ver [`../../../Analisis/IA-Mercado-Libre.md`](../../../Analisis/IA-Mercado-Libre.md))
> usa ambos.

---

## EVO-001 · Recuperación RAG semántica / híbrida

| Campo | Valor |
|---|---|
| Estado | `a-re-evaluar` |
| Prioridad | Alta |
| Esfuerzo | Media |
| Relación | Deriva `GAP-RAG-SEMANTIC` · [ADR-0006](../04-decisions/ADR-0006-rag-por-tenant.md) · [runtime-views](../01-architecture/04-runtime-views.md) |

### Contexto
Cada tenant necesita respuestas fundamentadas en su propia documentación. El diseño de origen
(`docs/05_arquitectura_tecnica/rag-spec_v1.0.md`) preveía **embeddings + similitud coseno (umbral 0.75)**.

### Estado actual (evidencia de código)
- 🟩 `RAGEngine.SearchRelevantChunksAsync` implementa **TF-IDF léxico en memoria** (stop-words ES, TF
  log-normalizado, IDF), `top-K = 5`, con *fallback* por substring. Puntúa **todos** los fragmentos del tenant por
  consulta (coste O(N)).
- 🟩 `KnowledgeService.UploadDocumentAsync` guarda `Vector_Embedding = null`; el helper `RAGEngine.SerializeEmbedding`
  es **código muerto**. La columna `sys_Fragmentos_Conocimiento.Vector_Embedding varbinary(MAX)` **ya existe**.
- **Conclusión:** hoy el RAG **no es semántico**; no capta sinónimos ni paráfrasis ("cargar plata" ≠ "ingresar
  dinero"), y no escala con el crecimiento de la base.

### Evolución propuesta
1. Al ingestar, calcular un **embedding por fragmento** con un modelo de *embeddings* y persistirlo en
   `Vector_Embedding`.
2. Al consultar, embeber la consulta y recuperar por **similitud coseno**.
3. Preferir **RAG híbrido**: TF-IDF (términos exactos, códigos, IDs) + semántico (significado) + *re-ranking*.
4. Devolver **cita de origen** (`Documento_Origen`) para trazabilidad.

### Impacto y decisiones abiertas
- **Dónde corre la búsqueda vectorial** (decisión central): en memoria, *vector store* dedicado, o soporte
  vectorial del motor. SQL Server de la versión actual no tiene tipo vectorial nativo.
- **Modelo de embeddings por tenant** (coherencia con la abstracción multi-proveedor — [ADR-0004](../04-decisions/ADR-0004-abstraccion-proveedor-ia.md)).
- Coste/latencia por *embeddings* (ingesta + consulta) y **re-embeber la KB existente**.

### Criterios de re-evaluación
- ¿La recuperación léxica está perdiendo relevancia medible (quejas, 👎, respuestas fuera de contexto)?
- ¿La KB por tenant creció al punto de que el escaneo O(N) impacta la latencia?
- ¿Se decide **converger el diseño** (implementar embeddings) o **actualizar `rag-spec`** para reflejar TF-IDF?

---

## EVO-002 · Function-calling / tools para datos dinámicos del usuario

| Campo | Valor |
|---|---|
| Estado | `propuesto` |
| Prioridad | Media-Alta |
| Esfuerzo | Alta |
| Relación | [ADR-0004](../04-decisions/ADR-0004-abstraccion-proveedor-ia.md) · estudio §B2/§D · [06-crosscutting](../01-architecture/06-crosscutting.md) |

### Contexto
El estudio de asistencia por IA muestra que los asistentes maduros combinan conocimiento estático (RAG) con
**datos dinámicos del usuario en vivo** (saldo, líneas, movimientos) y acciones. Referencia observada: el asistente
de Mercado Pago lee "tus líneas guardadas" y declara su alcance de datos (patrones P5/P6 del estudio).

### Estado actual (evidencia de código)
- 🟩 El contrato `IAIProvider` expone **solo** `Chat/Complete/Analyze/Summarize/Improve`; **no** hay definición de
  *tools* ni bucle de *tool-call*.
- 🟩 En `ChatService` + `PromptBuilder`, los datos dinámicos solo pueden entrar **inyectados a mano** en el *system
  prompt* o el historial; el modelo **no puede decidir ni solicitar** datos en vivo.
- **Conclusión:** IAConnect **no puede** hoy reproducir el comportamiento de "traer datos del usuario bajo demanda"
  de forma controlada por el modelo.

### Evolución propuesta
1. Declarar **tools** (funciones con *schema* JSON, ej.: `getLineasGuardadas(userId)`), que el LLM pueda invocar.
2. Extender la abstracción `IAIProvider` / `ChatRequest` para soportar **definición de tools** y el bucle
   llamada→ejecución→resultado→continuación, **normalizando** las APIs nativas de Claude/Gemini/OpenAI.
3. **Registro de tools por tenant** + implementaciones que llaman a los backends de negocio.
4. **Loop de ejecución** en `ChatService` con límites (máx. iteraciones, timeouts) y **auditoría**.
5. 🟦 Evaluar **MCP (Model Context Protocol)** como estándar de exposición de tools/datos.

### Impacto — seguridad (enlaza con estudio §D)
- La tool es el lugar correcto para **hacer cumplir la autorización**: se ejecuta con la identidad del JWT y
  devuelve **solo** los datos de ese usuario → evita el sobre-alcance y la fuga entre identidades.
- Es **más seguro** que volcar datos crudos en el prompt; habilita el patrón de **disclosure de alcance**.
- Aumenta la superficie de riesgo (*insecure output handling*, abuso de tools) → requiere **guardrails**.

### Criterios de re-evaluación
- ¿Hay casos de uso que **requieran datos en vivo** del usuario (no cubiertos por RAG estático)?
- ¿El asistente debe **ejecutar acciones** (además de informar)? Si sí, esto es prerrequisito.
- ¿Está resuelta la **autorización por operación e identidad** en los backends que las tools invocarían?
- ¿Conviene un contrato propio o adoptar **MCP** directamente?

---

## Referencias

- Estudio: [`Analisis-Asistencia-IA-ChatBotIA.md`](../../../Analisis/Analisis-Asistencia-IA-ChatBotIA.md) (§B2 integración, §C conocimiento, §D seguridad, §F tendencias).
- Caso de referencia: [`IA-Mercado-Libre.md`](../../../Analisis/IA-Mercado-Libre.md) (patrones P5/P6, §7 patrones reutilizables).
- Decisiones relacionadas: [ADR-0004](../04-decisions/ADR-0004-abstraccion-proveedor-ia.md), [ADR-0006](../04-decisions/ADR-0006-rag-por-tenant.md).
- Contexto de código (ia-db): [`04_proveedores-ia-y-rag.md`](../../../ia-db/indexes/04_proveedores-ia-y-rag.md), [`05_seguridad-y-multitenant.md`](../../../ia-db/indexes/05_seguridad-y-multitenant.md).
