# 06 — Administrator Guide · Asistencia sobre Turnos en GDA

> **Propósito.** Guía operativa del **administrador funcional** (referente de negocio) del asistente IA del caso *Turnos en GDA*: quién mantiene el conocimiento, cómo se redacta, cómo se releva el vocabulario del vecino, cómo se prueba un cambio y cómo se detecta un hueco de conocimiento.
> **Alcance.** Lo **específico del caso Turnos**. La metodología general de KB (crear RAG, subir documentos, editar fragmentos, administrar tenants y usuarios) está en [../Ng-IAServices/06-Administrator-Guide.md](../Ng-IAServices/06-Administrator-Guide.md) y **no se repite acá**.
> **Audiencia.** Referente funcional de Turnos (Municipio), coordinador de mesa de entradas / oficina de turnos, y el analista que hace de puente con el equipo técnico. **No** requiere saber programar.
> **Estado.** 🟨 Propuesta de diseño operativo. El asistente de Turnos **no está en producción**: hoy sólo existe una PoC del widget gateada a un DNI (🟩 `GDA.Core.Ciudadano/Components/Pages/Index.razor:126`). Todo procedimiento marcado 🟨 describe el régimen objetivo, no una práctica vigente.
> **Documentos hermanos.** [01-SAD](01-SAD.md) · [02-HLD](02-HLD.md) · [03-LLD](03-LLD.md) · [04-ADR](04-ADR.md) · [05-Operations-Guide](05-Operations-Guide.md) · **06-Administrator-Guide** · [07-Plan-Sprints-Capacitacion](07-Plan-Sprints-Capacitacion.md)

**Convención de marcas** (heredada de [Analisis-Asistencia-IA-ChatBotIA.md](../Antecedentes/Analisis-Asistencia-IA-ChatBotIA.md)):

| Marca | Significado |
|---|---|
| 🟩 | Hecho verificado en fuente (se cita ruta y, cuando aplica, línea). |
| 🟦 | Práctica de industria establecida. |
| 🟨 | Interpretación / inferencia / propuesta propia de este estudio. |
| No verificado | Afirmación que no pudo respaldarse; se declara como tal. |

---

## Tabla de contenidos

1. [Introducción, audiencia y responsabilidades](#1-introducción-audiencia-y-responsabilidades)
2. [Modelo mental: qué sabe y qué no sabe el asistente](#2-modelo-mental-qué-sabe-y-qué-no-sabe-el-asistente)
3. [Contenido de la KB del caso](#3-contenido-de-la-kb-del-caso)
4. [Cómo redactar el contenido del caso (BUENO vs MALO)](#4-cómo-redactar-el-contenido-del-caso-bueno-vs-malo)
5. [Gestión de sinónimos y lenguaje del usuario](#5-gestión-de-sinónimos-y-lenguaje-del-usuario)
6. [Tareas frecuentes paso a paso](#6-tareas-frecuentes-paso-a-paso)
7. [Ajuste del system prompt](#7-ajuste-del-system-prompt)
8. [Banco de preguntas de regresión](#8-banco-de-preguntas-de-regresión)
9. [Lectura de métricas y feedback: el ciclo de mejora](#9-lectura-de-métricas-y-feedback-el-ciclo-de-mejora)
10. [Diagnóstico: árbol de decisión ante "el asistente responde mal"](#10-diagnóstico-árbol-de-decisión-ante-el-asistente-responde-mal)
11. [Qué NO debe hacer el administrador](#11-qué-no-debe-hacer-el-administrador)
12. [Checklist periódico](#12-checklist-periódico)
13. [Trazabilidad de evidencia](#13-trazabilidad-de-evidencia)

---

## 1. Introducción, audiencia y responsabilidades

### 1.1 Por qué existe este rol

El asistente de Turnos **no tiene function-calling**: 🟩 un grep exhaustivo sobre `tool_use` / `tool_choice` / `function_call` en toda la solución IAConnect devuelve cero resultados. Y del lado de GDA, 🟩 el único endpoint REST de turnos es `POST Turnos/ProcesarRecordatorios` (sin autenticación, sólo dispara notificaciones) — 🟩 no existe API de consulta/alta/cancelación (`ia-db/indexes/02_apis-servicios.md §1`).

**Consecuencia directa para el administrador:** 🟨 en la Fase 1 del caso, *todo* lo que el asistente sabe proviene de dos lugares y de ningún otro:

1. El **system prompt** del tenant (`lut_Tenants.System_Prompt`, 🟩 `scripts/01_create_database.sql:31-53`).
2. Los **fragmentos de KB** recuperados por TF-IDF léxico (🟩 `RAGEngine.cs:34-120`).

No hay consulta en vivo a la base de turnos. Si el contenido no está escrito en la KB, el asistente **no lo sabe** — y en el peor caso, lo inventa. **La calidad del asistente ES la calidad de la KB.** Ese es el trabajo de este rol.

### 1.2 Reparto de responsabilidades

```mermaid
flowchart LR
  subgraph NEG["Negocio (esta guía)"]
    AF["Administrador funcional<br/>de Turnos"]
    RC["Referente Ciudadano<br/>(mesa de entradas)"]
    RF["Referente Funcionario<br/>(coord. de oficina)"]
  end
  subgraph TEC["Técnico (05-Operations-Guide)"]
    OPS["Operaciones IAConnect"]
    DEV["Desarrollo GDA"]
  end

  AF -->|redacta y publica| KB["KB tenant<br/>gda-turnos-ciudadano<br/>gda-turnos-funcionario"]
  AF -->|propone cambio| SP["System prompt"]
  RC -->|aporta vocabulario real| AF
  RF -->|aporta casos de mostrador| AF
  OPS -->|sube documento / rota keys / mira logs| KB
  OPS -->|aplica| SP
  DEV -->|cambia rutas, catálogo, UI| GDA["GDA.Core"]
  GDA -.->|"desincroniza"| KB
  AF -->|reporta desvío| DEV

  classDef n fill:#e8f5e9,stroke:#2e7d32
  classDef t fill:#e3f2fd,stroke:#1565c0
  class AF,RC,RF n
  class OPS,DEV t
```

| Responsabilidad                                               | Administrador funcional                                                              | Operaciones / Desarrollo                     |
| ------------------------------------------------------------- | ------------------------------------------------------------------------------------ | -------------------------------------------- |
| Redactar y mantener el contenido de la KB                     | ✅ **Dueño**                                                                          | Ejecuta la carga si el admin no tiene acceso |
| Mantener el diccionario de sinónimos                          | ✅ **Dueño** (§5)                                                                     | —                                            |
| Correr el banco de regresión (§8)                             | ✅ **Dueño**                                                                          | —                                            |
| Proponer cambios al system prompt                             | ✅ Propone                                                                            | ✅ Aplica y versiona                          |
| Subir documentos vía `POST /api/tenants/{tenantId}/knowledge` | Sólo si tiene rol `admin` (🟩 `KnowledgeController` es `[Authorize(Roles="admin")]`) | ✅                                            |
| Rotar API keys, tocar el DDL, ver métricas crudas en SQL      | ❌ **Nunca** (§11)                                                                    | ✅                                            |
| Cambiar rutas / catálogo / código de GDA                      | ❌                                                                                    | ✅                                            |

🟨 **Regla de oro del rol:** el administrador funcional **escribe**, no **configura**. Todo lo que sea infra, credenciales o esquema es de Operaciones.

### 1.3 Dos audiencias, dos tenants

🟨 El caso define **dos perfiles con permisos y necesidades distintas**, y este estudio propone **dos tenants separados** en IAConnect ([02-HLD](02-HLD.md)):

| | `gda-turnos-ciudadano` | `gda-turnos-funcionario` |
|---|---|---|
| Quién pregunta | Vecino, sin formación en el sistema | Operador de oficina, ya capacitado |
| Registro | Coloquial, voseo, "vos" | Técnico, nombres reales del sistema |
| Qué NO puede saber | Datos de otros vecinos, reglas internas, nombres de oficinas `Interno=1` (🟩 `lut_Oficinas_Turnos.Interno` bit → oficinas no publicables al vecino) | — (pero tampoco datos personales; ver §11) |
| Deep-links que emite | `/ciudadano/...` | `/Agenda`, `/Oficina`, `/BuscarCiudadano` |
| Dueño del contenido | Referente Ciudadano | Referente Funcionario |

Motivo de la separación: 🟩 `lut_Tenants` lleva un **único** `System_Prompt` por tenant y 🟩 `RAGEngine.SearchRelevantChunksAsync` recupera **todos los fragmentos del tenant** (`GetListByIdTenantAsync(tenantId)`, `RAGEngine.cs:34-120`) — no hay filtro por audiencia dentro de un tenant. 🟨 Un solo tenant mezclaría contenido de mostrador con contenido público en el mismo pool de recuperación: inaceptable.

---

## 2. Modelo mental: qué sabe y qué no sabe el asistente

Esta sección es **lo primero que hay que entender**. Casi todos los errores de administración vienen de asumir que el asistente puede algo que no puede.

### 2.1 Cómo llega una pregunta a una respuesta

```mermaid
sequenceDiagram
  autonumber
  actor V as Vecino
  participant W as Widget (GDA.Core.Ciudadano)
  participant API as AIController<br/>/api/ai/{tenantId}/chat
  participant RAG as RAGEngine (TF-IDF)
  participant KB as sys_Fragmentos_Conocimiento
  participant PB as PromptBuilder
  participant LLM as Proveedor IA

  V->>W: "quiero sacar turno pa'l registro"
  W->>API: POST chat {message, sessionId}
  API->>RAG: SearchRelevantChunks(tenantId, message)
  RAG->>KB: GetListByIdTenant(tenantId)  — TODOS los fragmentos
  KB-->>RAG: N fragmentos
  Note over RAG: tokeniza, descarta stop-words<br/>y tokens de ≤2 chars, computa TF-IDF
  RAG-->>API: top-5 fragmentos por score léxico
  API->>PB: BuildSystemPrompt(tenant, query, chunks, history)
  PB-->>API: SystemPrompt + [CONTEXTO RELEVANTE] + [HISTORIAL] + [CONSULTA]
  API->>LLM: chat(system, messages)
  LLM-->>API: texto
  API-->>W: respuesta
  W-->>V: burbuja + link
```

🟩 Verificado en `RAGEngine.cs:34-120`, `PromptBuilder.cs:10-55`, `ChatService.cs:46-189`.

### 2.2 El punto que **más** afecta al administrador: el RAG es **léxico**, no semántico

🟩 Hallazgo verificado y central: aunque el esquema define `Vector_Embedding varbinary(MAX)` y el documento de origen `rag-spec_v1.0.md` describe similitud coseno con threshold 0.75, **el código no calcula embeddings**. `KnowledgeService` persiste siempre `VectorEmbedding = null` (`KnowledgeService.cs:75`), `SerializeEmbedding` es código muerto que nadie invoca (`RAGEngine.cs:122-127`), y la recuperación es TF-IDF sobre palabras.

🟨 **Traducción al lenguaje del administrador:**

> El asistente **no entiende que "registro" y "Licencia de Conducir" son lo mismo**. Sólo encuentra un fragmento si el vecino usa **palabras que están literalmente escritas en ese fragmento** (o un substring de ellas).

Esto invierte la regla habitual de redacción de KB. 🟦 En un RAG semántico se recomienda escribir prosa limpia y dejar que el modelo generalice. 🟨 **Acá NO**: hay que **sembrar en el texto todas las palabras con las que la gente puede preguntar**. Por eso §5 (sinónimos) no es un adorno: es el corazón operativo de este caso.

Detalles del tokenizador que condicionan la redacción (🟩 `RAGEngine.cs:14-24, 34-120`):

| Comportamiento del motor | Marca | Qué implica al redactar |
|---|---|---|
| Descarta tokens de longitud **≤ 2 caracteres** | 🟩 | Palabras como "DNI" (3, sobrevive) sí; pero "ID", "vtv" corto, "CI" **no matchean**. No confíes en siglas de 2 letras. |
| Descarta ~57 stop-words en español + 11 en inglés (`de`, `la`, `para`, `que`, `como`…) | 🟩 | Una consulta como "¿cómo saco un turno?" aporta casi sólo `saco` y `turno`. Los verbos coloquiales importan. |
| Fallback por **substring**: si `tf==0` pero el término aparece como substring del contenido, fuerza `tf=1` | 🟩 | "licenci" matchea "licencia". Ayuda con plurales y truncamientos, pero **no** con sinónimos. |
| IDF: `log(totalDocs / (1 + docsWithTerm)) + 1` | 🟩 | Una palabra que está en **todos** los fragmentos pierde poder discriminante. No repitas "turno" 40 veces en cada fragmento: lo degradás como término de búsqueda. |
| `topK = 5` por defecto | 🟩 | Sólo los **5** fragmentos de mayor score llegan al modelo. Si tenés 8 fragmentos que compiten por la misma pregunta, 3 se pierden. Favorece fragmentos **pocos y específicos**. |
| Recupera **todos** los fragmentos del tenant en cada request, sin caché ni paginación | 🟩 (🟨 riesgo O(N·M)) | La KB debe mantenerse **acotada**. No es un archivo histórico: es un índice de respuestas vivas. |
| **No** normaliza acentos | 🟩 (no hay `Normalize`/`RemoveDiacritics` en `Tokenize`) | Crítico: 🟩 los datos reales van **sin tildes** ("Clinica Medica", `01-sacar-turno-licencia-conducir...spec.ts:11,55`) pero el vecino escribe **con** tilde. Hay que escribir **ambas** formas (§5.4). |

### 2.3 Qué NO puede hacer el asistente (Fase 1)

| El asistente NO puede… | Por qué (evidencia) |
|---|---|
| Ver los turnos de un vecino | 🟩 No hay function-calling (grep verificado) ni API REST de consulta (🟩 `02_apis-servicios.md §1`). |
| Sacar, anular ni reprogramar un turno | Ídem. Además 🟩 **la reprogramación no existe en GDA**: grep global por `reprogram` sobre `*.cs`/`*.razor` en toda la solución = **0 hits**. |
| Decir si hay disponibilidad para mañana | 🟩 Los slots son filas pre-creadas en `sys_Turnos` (~15.985) consultadas por SP `Id_Oficina_Proximos`; el asistente no las consulta. |
| Saber el nombre exacto de un trámite dado de alta ayer | 🟩 `lut_MotivosTurnos` no se sincroniza automáticamente con la KB. La KB es **manual**. Ver §6.3. |
| Entender que "el registro" = "Licencia de Conducir" | 🟩 RAG léxico (§2.2) + 🟩 **no existe ninguna tabla ni columna de alias/sinónimos/keywords en el área turnos** (grep sobre los 27 archivos del diccionario: 0 hits en `turnos.md`). |

### 2.4 Qué SÍ puede hacer, y muy bien

🟨 Bien administrado, el asistente resuelve el problema real del caso: **el vecino no sabe cómo se llama el trámite**.

- Mapear lenguaje coloquial → nombre real del motivo, y ofrecer opciones si hay ambigüedad. 🟦 Es el patrón de **disclosure de alcance** + **divulgación progresiva** observado en [IA-Mercado-Libre.md](../Antecedentes/IA-Mercado-Libre.md).
- Entregar el **deep-link** correcto: 🟩 `/ciudadano/TurnosLugar?m={IdMotivo}` aterriza en el trámite **con sus requisitos** (🟩 `TurnosLugar.razor.cs:33-34` renderiza `lut_MotivosTurnos.Comentario` con `MarkupString` si `MostrarComentario=1`).
- Explicar requisitos, topes, penalizaciones y mensajes de concurrencia con el **texto literal del sistema** (🟩 `TurnosService.cs:148-190, 197-360`).
- Hacer **hand-off** honesto: "esto no lo puedo hacer yo, entrá acá".

```mermaid
stateDiagram-v2
  [*] --> Escucha
  Escucha --> Identificado: la consulta matchea un motivo
  Escucha --> Ambiguo: matchea 2+ motivos
  Escucha --> SinMatch: no matchea nada
  Identificado --> EntregaLink: nombre real + requisitos + deep-link
  Ambiguo --> Desambigua: ofrece opciones (máx. 3)
  Desambigua --> Identificado
  Desambigua --> SinMatch: ninguna sirve
  SinMatch --> HandOff: "no tengo ese trámite; mirá el listado / consultá en la oficina"
  EntregaLink --> [*]
  HandOff --> [*]
  note right of SinMatch
    🟨 Nunca inventar un motivo.
    Ver §7 y el banco R-12.
  end note
```

---

## 3. Contenido de la KB del caso

### 3.1 Principio de organización

🟨 **Un documento = un tema = pocos fragmentos.** Recordá (§2.2) que sólo entran **5** fragmentos al prompt y que cada uno pesa ~400 **palabras**.

🟩 Precisión importante sobre el troceo, verificada en fuente: `KnowledgeService` declara `const int ChunkSizeTokens = 400` y `OverlapTokens = 50` (`KnowledgeService.cs:16-17`), pero `SplitIntoChunks` **no tokeniza**: hace `text.Split(new[]{' ','\n','\r','\t'}, RemoveEmptyEntries)` y avanza `step = 350` **palabras** (`:103-121`). 🟨 Es decir: la unidad real es la **palabra**; 400 palabras ≈ 520-600 tokens en español. La constante está mal nombrada.

🟨 **Regla práctica para el administrador:** escribí documentos de **~300-350 palabras por tema**. Así cada tema cae limpio en **un** fragmento y no se parte por la mitad. Un documento de 2.000 palabras se corta en ~6 fragmentos ciegamente, y el corte puede caer justo entre la pregunta y su respuesta.

### 3.2 Inventario propuesto — tenant `gda-turnos-ciudadano`

🟨 Propuesta de este estudio. La columna "Fuente de verdad" indica **de dónde se copia el dato**, no de dónde lo lee el asistente (que lee sólo la KB).

| # | Documento | Contenido | Fuente de verdad | Dueño | Revisión |
|---|---|---|---|---|---|
| C-01 | `catalogo-tramites.md` | Los 39 motivos activos: nombre real + tipo + oficinas + deep-link. Un bloque por motivo. | 🟩 `lut_MotivosTurnos` (39), `lut_TiposTurnos` (14), `lut_MotivosTurnos_Oficinas` (72) | Ref. Ciudadano | **Mensual** + ante toda alta/baja/renombre |
| C-02 | `diccionario-sinonimos.md` | Cómo llama la gente a cada trámite (§5). **El documento más importante del caso.** | Relevamiento propio (§5.2) | Ref. Ciudadano | **Quincenal** (alimentado por §9) |
| C-03 | `requisitos-por-tramite.md` | Qué papeles llevar, por motivo. | 🟩 `lut_MotivosTurnos.Comentario` (HTML crudo, varchar 3000) | Ref. Ciudadano | Mensual |
| C-04 | `como-sacar-turno.md` | El paso a paso del vecino. | 🟩 Wizard de 7 pasos `EntregaTurnosComponent` + rutas | Ref. Ciudadano | Trimestral |
| C-05 | `mis-turnos-cancelar.md` | Ver turnos, cancelar, **y que NO hay reprogramación**. | 🟩 `/Turnos`, `/TurnoDetalle?Id=`; 🟩 grep `reprogram` = 0 hits | Ref. Ciudadano | Trimestral |
| C-06 | `reglas-topes-y-ausencias.md` | Tope por período; penalización por ausentismo. | 🟩 `TurnosService.ValidarUsuario` :197-278 + `lut_Oficinas_Turnos_Validaciones` (3 filas) | Ref. Funcionario | **Ante cambio de parámetro** |
| C-07 | `mensajes-y-errores.md` | Qué significa cada mensaje que ve el vecino. | 🟩 `TurnosService.ValidarTurnoDisponible` :148-190 | Ref. Ciudadano | Trimestral |
| C-08 | `recordatorios-y-avisos.md` | Push OneSignal + email; flags de recordatorio. | 🟩 `TurnosService.procesarRecordatorios` :44-100 | Ref. Ciudadano | Semestral |
| C-09 | `cuenta-vecino-digital.md` | Registro por DNI, recupero de clave. | 🟩 `docs/pieces/ciudadano/README.md §Autenticación` | Ref. Ciudadano | Semestral |
| C-10 | `limites-del-asistente.md` | Qué NO puede hacer y a dónde derivar. | Este documento §2.3 | Admin funcional | Ante cada cambio de alcance |

### 3.3 Inventario propuesto — tenant `gda-turnos-funcionario`

| # | Documento | Contenido | Fuente de verdad | Dueño | Revisión |
|---|---|---|---|---|---|
| F-01 | `agenda-operacion-diaria.md` | Navegar por fecha, imprimir, marcar presente (**irreversible**), anular. | 🟩 `Agenda.razor.cs:146-250`, `Agenda.razor:114,279,329` | Ref. Funcionario | Trimestral |
| F-02 | `elegir-oficina.md` | `/Oficina` obligatorio tras login; cómo cambiar. | 🟩 `AuthManagerTurnos.cs:120-135` + `docs/pieces/backoffice-turnos/README.md` | Ref. Funcionario | Semestral |
| F-03 | `reglas-que-no-puedo-saltear.md` | El funcionario **no** puede saltear topes. | 🟩 `ValidarUsuario_Funcionario` :280-360 (mismos topes, mensajes en 3ª persona) | Ref. Funcionario | Ante cambio de parámetro |
| F-04 | `abm-catalogo.md` | Alta de tipo/motivo/lugar; dónde se cargan los requisitos (`Comentario` + `MostrarComentario`). | 🟩 rutas `/TurnosTipo`, `/TurnosMotivo`, `/TurnosLugar` del BackOffice | Ref. Funcionario | Semestral |
| F-05 | `sin-reprogramacion.md` | No existe; anular + otorgar de nuevo. | 🟩 grep `reprogram` = 0 hits | Ref. Funcionario | Ante cambio |
| F-06 | `sin-informes.md` | 🟨 No hay página de informes de turnos; lo más cercano es "Imprimir Turnos" de `/Agenda`. | 🟩 `Agenda.razor.cs:146` + 🟨 `ia-db/indexes/06_generacion-v2.md §2.1` | Ref. Funcionario | Semestral |
| F-07 | `buscar-ciudadano.md` | `/BuscarCiudadano`, `/Ciudadano`, turnos del vecino. | 🟩 `CiudadanoTurnosComponent.razor.cs:58-67` | Ref. Funcionario | Semestral |
| F-08 | `limites-del-asistente-bo.md` | Qué no puede hacer; **nunca** pedirle datos personales. | §11 | Admin funcional | Ante cambio |

### 3.4 Estructura de archivos del administrador

🟨 Propuesta: la KB se mantiene como archivos versionados en Git y se **publica** a IAConnect; nunca se edita "sólo en la base".

```text
GDA-Turnos-KB/
├── README.md                        # cómo publicar, quién es dueño de qué
├── ciudadano/                       # → tenant gda-turnos-ciudadano
│   ├── C-01-catalogo-tramites.md
│   ├── C-02-diccionario-sinonimos.md
│   ├── C-03-requisitos-por-tramite.md
│   ├── C-04-como-sacar-turno.md
│   ├── C-05-mis-turnos-cancelar.md
│   ├── C-06-reglas-topes-y-ausencias.md
│   ├── C-07-mensajes-y-errores.md
│   ├── C-08-recordatorios-y-avisos.md
│   ├── C-09-cuenta-vecino-digital.md
│   └── C-10-limites-del-asistente.md
├── funcionario/                     # → tenant gda-turnos-funcionario
│   ├── F-01-agenda-operacion-diaria.md
│   ├── ...
│   └── F-08-limites-del-asistente-bo.md
├── prompts/
│   ├── system-prompt-ciudadano.md   # versionado; se aplica vía Operaciones
│   └── system-prompt-funcionario.md
└── regresion/
    ├── banco-ciudadano.md           # §8
    ├── banco-funcionario.md
    └── resultados/
        └── 2026-07-16-r1.md         # una corrida = un archivo
```

⚠ 🟩 **Advertencia de ingesta que el administrador DEBE conocer:** `UploadDocumentAsync` **no borra los fragmentos previos** del mismo `Documento_Origen` — no hay dedupe. Subir dos veces el mismo archivo **duplica los fragmentos** (`KnowledgeService.cs:34-101`). 🟨 Procedimiento obligatorio: **borrar antes de resubir** (pedir a Operaciones si no tenés rol `admin`). Ver §6.1.

🟩 Formatos aceptados: `.pdf` (vía UglyToad.PdfPig), `.txt`, `.md`, `.html`/`.htm`, `.csv`. Cualquier otro → `ArgumentException "Formato de archivo no soportado"` → HTTP 400 (`KnowledgeService.cs:34-101` + `GlobalExceptionMiddleware.cs:32-41`). 🟨 **Usá `.md`**: el PDF pierde estructura al extraer texto por concatenación de `page.Text`.

---

## 4. Cómo redactar el contenido del caso (BUENO vs MALO)

### 4.1 Las siete reglas del caso Turnos

| # | Regla | Fundamento |
|---|---|---|
| R1 | **Sembrá el vocabulario del vecino en el propio texto.** | 🟩 RAG léxico (§2.2). Sin la palabra, no hay match. |
| R2 | **Un tema por documento, ~300-350 palabras.** | 🟩 Chunk de 400 palabras / paso 350 (`KnowledgeService.cs:103-121`). |
| R3 | **Escribí acentuado Y sin acentuar** cuando el dato real va sin tilde. | 🟩 Datos sin tildes ("Clinica Medica"); 🟩 tokenizador no normaliza. |
| R4 | **Copiá los mensajes literales del sistema**, entre comillas. | 🟩 `TurnosService.cs:148-190` — el vecino los ve en pantalla y los va a pegar en el chat. |
| R5 | **Poné el deep-link completo y exacto**, como lo emite el código. | 🟩 `TurnoDetalle?Id=` (I mayúscula), `turno?id=&m=&o=`, `TurnoAsignado?id=` (`Turno.razor.cs:52-57`, `TurnoDetalle.razor.cs:38-41`). |
| R6 | **Decí explícitamente lo que NO se puede.** | 🟩 No hay reprogramación (0 hits). El silencio se llena con alucinación. |
| R7 | **Nunca pongas datos personales ni credenciales.** | Ver §11. |

### 4.2 Ejemplo completo — Catálogo (C-01), un motivo

#### ❌ MALO

```markdown
## Licencia de Conducir

Trámite disponible. Consultar en la oficina correspondiente.
Ver más información en el portal.
```

**Por qué es malo:**
- 🟨 No contiene ninguna palabra con la que el vecino va a preguntar ("registro", "carnet", "manejar"). Un vecino que escribe *"quiero sacar el registro"* aporta los términos `quiero`, `sacar`, `registro` — 🟩 `quiero`/`sacar` no están y `registro` tampoco: **score 0, el fragmento no se recupera**.
- No hay link → el asistente lo inventa o no lo da.
- "Consultar en la oficina correspondiente" es exactamente lo que el vecino ya no quería hacer.
- 20 palabras: desperdicia un fragmento entero (capacidad 400).

#### ✅ BUENO

```markdown
## Trámite: Licencia de Conducir

**Nombre exacto en el sistema:** Licencia de Conducir
**Categoría (tipo de turno):** Licencias
**También conocido como:** registro, carnet de conducir, carnet de manejar,
licencia de manejar, licencia de conducir, licencia de conducción, el registro
de auto, registro de moto, sacar el registro, renovar el registro, renovación
de licencia, licencia nueva, licencia por primera vez, brevete, carné.

**Dónde se atiende:** Clinica Medica (Clínica Médica) — la oficina se llama así
en el sistema, sin tildes.

**Enlace para sacar turno:**
https://<host>/ciudadano/TurnosLugar?m=<IdMotivo>
Ese enlace muestra la lista de lugares disponibles y, arriba, los requisitos
del trámite.

**Qué necesitás antes de empezar:** tener cuenta de Vecino Digital (te registrás
con tu DNI) y a mano tu nombre, apellido, DNI, celular y correo electrónico:
esos cinco datos son obligatorios para confirmar el turno.

**Requisitos del trámite:** los ves en la misma pantalla del enlace de arriba,
en el recuadro de arriba de la lista de lugares.

**Ojo:** una vez que elegís el horario tenés 5 minutos para completar tus datos.
Si tardás más, el turno vuelve a quedar libre para otra persona.
```

**Por qué es bueno:**
- 🟨 La línea "También conocido como" es un **campo de siembra léxica**: le da al TF-IDF los términos que el vecino realmente escribe. Es la compensación manual de 🟩 "no existe tabla de alias en el área turnos".
- 🟩 Escribe "Clinica Medica" **y** "Clínica Médica" (R3), porque el dato real va sin tilde (`01-sacar-turno-licencia-conducir...spec.ts:11,55`) y el vecino escribe con tilde.
- 🟩 El deep-link es el útil: `/ciudadano/TurnosLugar?m={IdMotivo}` aterriza en el trámite **con requisitos** (`TurnosLugar.razor.cs:33-34`).
- 🟩 Enumera los 5 campos obligatorios reales (`EntregaTurnosComponent.razor.cs:713-752`: Nombre, Apellido, Motivo, Celular, Email).
- 🟩 Explica la reserva blanda de 5 minutos (`EntregaTurnosComponent.razor.cs:284-285`) en lenguaje de vecino, sin decir "Fecha_Reserva".
- ~200 palabras: entra holgado en un fragmento.

### 4.3 Ejemplo completo — Reprogramación (C-05)

#### ❌ MALO

```markdown
## Modificar un turno

Si necesitás cambiar tu turno, gestionalo desde el sistema.
```

**Por qué es malo:** 🟨 ambiguo hasta el daño. "Gestionalo desde el sistema" le sugiere al modelo que **existe** una función de cambio. 🟩 No existe: grep global por `reprogram` sobre `*.cs`/`*.razor` = **0 hits**. El modelo va a completar el hueco inventando un botón "Reprogramar" que el vecino nunca va a encontrar. Éste es el arquetipo de la alucinación por omisión.

#### ✅ BUENO

```markdown
## ¿Puedo cambiar la fecha u hora de mi turno?

**No se puede reprogramar, reagendar, cambiar, mover, modificar, correr ni
postergar un turno ya sacado.** El sistema no tiene esa función: no hay botón
de "reprogramar" ni de "cambiar fecha" en ninguna pantalla.

**Lo que sí podés hacer** es cancelar el turno que tenés y sacar uno nuevo:

1. Entrá a "Mis turnos": https://<host>/ciudadano/Turnos
2. Tocá el turno que querés cambiar. Se abre el detalle:
   https://<host>/ciudadano/TurnoDetalle?Id=<IdTurno>
3. Cancelalo desde ahí.
4. Volvé a sacar turno normalmente para el trámite que necesitás.

**Importante antes de cancelar:** fijate que haya horarios libres para la nueva
fecha. Si cancelás y después no hay disponibilidad, perdiste el que tenías.

**Importante 2:** cancelar no es lo mismo que faltar. Si cancelás, no te
penalizan. Si no cancelás y no vas, puede contarte como ausencia y quedarte
bloqueado por un tiempo para sacar turnos (ver "Faltas y penalizaciones").
```

**Por qué es bueno:**
- 🟨 Siembra **ocho** verbos coloquiales del mismo concepto ("reprogramar, reagendar, cambiar, mover, modificar, correr, postergar") — cualquiera de ellos recupera el fragmento (R1).
- 🟩 Da el camino real (`/Turnos` → `/TurnoDetalle?Id=`, verificado en las rutas del portal) con el `Id` capitalizado tal como lo emite el código (R5).
- 🟨 Agrega la advertencia operativa que un humano de mostrador daría, y que el sistema no da.
- 🟩 Conecta con la penalización real por ausentismo (`TurnosService.cs:197-278`).

### 4.4 Ejemplo completo — Mensaje de error (C-07)

#### ❌ MALO

```markdown
Si aparece un error de reserva, reintentar.
```

#### ✅ BUENO

```markdown
## "Otro usuario esta reservando este turno. Volvé mas tarde o elegí otro."

Si te aparece este mensaje, el horario que elegiste lo está tomando otra
persona **en este mismo momento**. Cuando alguien elige un horario, el sistema
se lo guarda **5 minutos** mientras completa sus datos. Durante esos 5 minutos
nadie más lo puede tomar.

**Qué hacer:** elegí otro horario de la lista, o esperá unos minutos y refrescá
la página. Si la otra persona no confirma en 5 minutos, el horario vuelve a
quedar libre.

## "El turno acaba de ser tomado. Volvé a intentar con otro horario."

Alguien confirmó ese horario justo antes que vos. Ya no está disponible.
**Qué hacer:** elegí otro horario o probá con otro día.

## "Horario de turno pasado."

Estás intentando tomar un horario que ya pasó. Suele salir si dejaste la
pantalla abierta mucho tiempo. **Qué hacer:** refrescá la página y elegí un
horario futuro.

## No me carga la lista de trámites / la pantalla sale vacía

Si entrás y no ves ningún trámite, ninguna oficina o ningún horario, y la
pantalla queda en blanco sin mensaje, probablemente sea un problema temporal
del sistema. **Qué hacer:** refrescá, probá en unos minutos, y si sigue igual
reportalo a la oficina de turnos. No es un problema de tu cuenta.
```

**Por qué es bueno:**
- 🟩 Usa los **textos literales** del sistema como títulos, incluyendo la ortografía real ("esta reservando" sin tilde, "mas tarde" sin tilde) — `TurnosService.cs:148-190`. El vecino copia y pega el mensaje: si el texto está literal en la KB, el match TF-IDF es casi perfecto (R4).
- 🟨 El último bloque cubre un comportamiento real: 🟩 las páginas de turnos del portal tienen `catch (Exception ex) { }` **vacío** en `OnInitializedAsync` (`Turnos.razor.cs:40-43`, `TurnosTipo.razor.cs:14-17`, `TurnosMotivo.razor.cs:30-33`, `TurnosLugar.razor.cs:37-40`) → si falla la BD, el vecino ve una pantalla vacía **sin mensaje**. La KB debe cubrir ese hueco de UX.

### 4.5 Ejemplo completo — Contenido de funcionario (F-01)

#### ❌ MALO

```markdown
## Presente
Botón para marcar presente al ciudadano.
```

#### ✅ BUENO

```markdown
## Marcar a una persona como presente en la agenda

En la pantalla /Agenda, cada turno del día tiene el botón **Presente**. Al
tocarlo, el sistema pide confirmación con este texto:

"¿Estás seguro de que querés marcar esta persona como presente? Una vez
realizado no podrás anular el presentismo."

**La acción es IRREVERSIBLE.** No hay botón para desmarcar, revertir, deshacer
ni corregir el presentismo. Si marcaste presente a la persona equivocada, no
se puede arreglar desde la aplicación: hay que reportarlo al área de sistemas.

**Antes de tocar Presente:** verificá el nombre y el DNI de la fila.

## Anular un turno desde la agenda

En /Agenda también está el botón **Anular**, que da de baja el turno del
vecino. También podés anular desde la ficha del ciudadano (/BuscarCiudadano →
/Ciudadano).

**No hay reprogramación:** si el vecino quiere cambiar de día, tenés que anular
y otorgarle uno nuevo desde /Turno. No existe función de cambiar la fecha.
```

**Por qué es bueno:** 🟩 usa el texto de confirmación literal y el hecho de irreversibilidad (`Agenda.razor.cs:146-250`, `Agenda.razor:114,279,329`), y 🟨 traduce la irreversibilidad en una **instrucción de conducta** ("verificá antes"), que es lo que un operador necesita.

### 4.6 Anti-patrón crítico: no uses los delimitadores del prompt

🟩 `PromptBuilder` arma el prompt con delimitadores en corchetes mayúsculas — `[CONTEXTO RELEVANTE]`, `[HISTORIAL DE CONVERSACIÓN]`, `[CONSULTA DEL USUARIO]` — y vuelca cada fragmento como `Fragmento {i+1}: "{Contenido}"` **entre comillas dobles y sin escapado** (`PromptBuilder.cs:10-55`).

🟨 **Implicancia directa:** si un documento de la KB contiene la cadena `[CONSULTA DEL USUARIO]`, o `Fragmento 3:`, o comillas dobles desbalanceadas en abundancia, puede **confundir los límites del prompt**. Es una superficie de prompt-injection vía documento subido.

**Regla:** 🟨 en los documentos de la KB, **nunca** escribas texto entre corchetes en mayúsculas, ni las palabras `Fragmento N:`, `Role:`, `User:`, `Assistant:` al inicio de línea. Usá `**negrita**` o `##` para destacar.

---

## 5. Gestión de sinónimos y lenguaje del usuario

> **Esta es la sección central del caso.** El problema que el usuario planteó textualmente es: *"un ciudadano podría consultar si hay turno para un trámite específico y el chatbot le podría indicar que existe ese trámite **o en realidad se llama diferente** e indicarle opciones"*. Todo este capítulo existe para resolver esa frase.

### 5.1 Por qué el sistema no ayuda en nada acá

🟩 **Hecho verificado, y es duro:** no existe ninguna tabla ni columna de alias, sinónimos, keywords o etiquetas en el área turnos. `lut_TiposTurnos` y `lut_MotivosTurnos` sólo tienen `Descripcion` como texto de nombre. Un grep sobre los 27 archivos del diccionario de datos por `alias|sinonim|keyword|etiqueta|tag` sólo devuelve `lut_MotivosIncidente_Etiquetas` / `sys_Incidentes_Etiquetas` (dominio **incidentes**) y `CBU_Alias` (dominio **compras**) — **0 hits en turnos**.

🟩 Y el motor de recuperación es léxico (§2.2): no puede inferir que "registro" ≈ "Licencia de Conducir".

🟨 **Conclusión inevitable:** el mapeo *nombre coloquial del vecino → nombre real del motivo* **debe resolverlo el asistente con un diccionario propio**, y ese diccionario **es un documento de la KB que mantiene el administrador funcional a mano**. No hay atajo técnico. Este es el activo de mayor valor del caso — y 🟨 **el más reutilizable** para los próximos casos (Multas, Habilitaciones, Reclamos): el mismo procedimiento se aplica tal cual.

```mermaid
flowchart TD
  A["Vecino escribe:<br/>'turno pal registro'"] --> B{"¿'registro' aparece<br/>literalmente en algún<br/>fragmento de la KB?"}
  B -->|"No hay diccionario"| C["TF-IDF: score 0<br/>en el fragmento correcto"]
  C --> D["Se recuperan 5 fragmentos<br/>irrelevantes o ninguno"]
  D --> E["❌ 'No encontré ese trámite'<br/>o peor: alucina uno"]
  B -->|"C-02 tiene<br/>'registro → Licencia de Conducir'"| F["TF-IDF: score alto<br/>en el fragmento del diccionario<br/>Y en el del catálogo"]
  F --> G["✅ 'Creo que buscás<br/>Licencia de Conducir.<br/>Acá está el link'"]

  style E fill:#ffebee,stroke:#c62828
  style G fill:#e8f5e9,stroke:#2e7d32
```

### 5.2 Procedimiento de relevamiento (cómo se construye el diccionario)

🟨 Propuesta de este estudio. Cinco fuentes, en orden de valor:

```mermaid
flowchart LR
  F1["1. Mostrador<br/>(operadores)"] --> POOL[("Pool de<br/>candidatos")]
  F2["2. Consultas<br/>reales del chat<br/>(sys_Mensajes)"] --> POOL
  F3["3. Buscador del<br/>portal / call center"] --> POOL
  F4["4. Redes y<br/>reclamos"] --> POOL
  F5["5. Sesión de<br/>lluvia de ideas<br/>con el equipo"] --> POOL
  POOL --> VAL{"Validación:<br/>¿mapea a UN<br/>motivo real?"}
  VAL -->|Sí| PUB["C-02<br/>diccionario-sinonimos.md"]
  VAL -->|"Mapea a varios"| AMB["Lista de<br/>desambiguación"]
  AMB --> PUB
  VAL -->|"No mapea a nada"| DESC["Descartar<br/>o crear FAQ<br/>de 'no existe'"]
```

| Fuente | Cómo se releva | Frecuencia | Marca |
|---|---|---|---|
| **1. Mostrador** | Pedile a 3-5 operadores de la oficina que anoten durante **2 semanas** la frase textual con la que el vecino pide cada trámite. Es la fuente de oro: es lenguaje real, oral, sin autocensura. | Al arrancar + anual | 🟨 |
| **2. Consultas reales del chat** | 🟩 `sys_Mensajes` guarda `Contenido` de cada mensaje con `Rol='user'` (`scripts/01_create_database.sql:58-196`). Operaciones exporta las consultas del período; el administrador las lee. **El administrador no consulta SQL** (§11): pide el export. | Quincenal | 🟨 |
| **3. Buscador del portal / call center** | Términos más buscados / motivos de llamada. | Trimestral | 🟨 (No verificado que exista instrumentación de búsqueda) |
| **4. Redes y reclamos** | Cómo la gente nombra el trámite cuando se queja. | Trimestral | 🟨 |
| **5. Lluvia de ideas** | Sentate con el equipo y por cada motivo preguntá: *"¿de cuántas formas distintas te lo piden?"*. Incluí errores de tipeo frecuentes y regionalismos. | Al dar de alta un motivo | 🟨 |

### 5.3 Formato del documento C-02

🟨 Propuesta. Un bloque por motivo, corto, denso en términos.

```markdown
# Cómo llama la gente a cada trámite

Si el vecino usa cualquiera de estas palabras, se está refiriendo al trámite
que figura como "Nombre en el sistema".

## registro / carnet / licencia → Licencia de Conducir
El vecino dice: registro, el registro, sacar el registro, renovar el registro,
carnet, carné, carnet de conducir, carnet de manejar, carné de manejar,
licencia, licencia de manejar, licencia de conducción, permiso de conducir,
brevete, quiero manejar, para el auto, para la moto, registro de moto,
registro de auto, licencia profesional, licencia por primera vez, primera
licencia, renovacion, renovación.
Nombre en el sistema: **Licencia de Conducir**

## médico / clínica / apto → Clinica Medica
El vecino dice: medico, médico, doctor, clinica, clínica, clinica medica,
clínica médica, apto medico, apto médico, revisación, revisacion, chequeo,
examen medico, examen médico, psicofisico, psicofísico, control de salud,
turno con el medico.
Nombre en el sistema: **Clinica Medica** (así, sin tildes)
Ojo: si el vecino pide "apto médico para el registro", en general necesita
primero Clinica Medica y después Licencia de Conducir. Preguntale para qué lo
necesita antes de darle un solo link.
```

**Puntos de diseño del formato** (🟨):
- El título del bloque lleva **los coloquialismos primero** y el nombre real después: así el título mismo aporta términos al TF-IDF.
- 🟩 Incluye **con y sin tilde** de cada término (`medico`/`médico`, `clinica`/`clínica`) porque el tokenizador no normaliza acentos y los datos reales van sin tildes.
- Incluye **errores de tipeo frecuentes** y **regionalismos** ("brevete").
- La nota final del segundo bloque es una **regla de desambiguación**: le enseña al modelo a **preguntar** en vez de adivinar. 🟦 Es el patrón de divulgación progresiva del antecedente [IA-Mercado-Libre.md](../Antecedentes/IA-Mercado-Libre.md).
- ⚠ 🟩 Los términos de **≤ 2 caracteres se descartan** (`RAGEngine.cs` filtra `length <= 2`): no pongas "CI", "vt", "id" esperando que matcheen.

### 5.4 La regla de los acentos, explicada

🟩 Evidencia: los nombres reales en homologación van **sin tildes** — motivo «Licencia de Conducir», motivo «Clinica Medica», oficina «Clinica Medica» (`01-sacar-turno-licencia-conducir-restaurando-usuario.spec.ts:11,55`, `02-...spec.ts:11,55`). 🟩 El tokenizador de `RAGEngine.Tokenize` hace lowercase y split por separadores, **sin** normalización de diacríticos.

🟨 Por lo tanto: `"médica"` y `"medica"` son **tokens distintos**. El fallback por substring 🟩 (`si tf==0 pero el término aparece como substring del contenido, fuerza tf=1`) **tampoco** salva esto: "médica" no es substring de "medica".

**Regla operativa:** 🟨 en el diccionario C-02, **cada término que lleve tilde se escribe dos veces**, con y sin. Es feo de leer, pero el documento C-02 no está hecho para que lo lean humanos: está hecho para que lo encuentre el buscador léxico. (Los documentos que **sí** se leen — C-01, C-03 — se escriben bien acentuados, con la forma sin tilde sólo donde reproducen el dato real del sistema.)

### 5.5 Mantenimiento del diccionario

| Disparador | Acción | Plazo |
|---|---|---|
| Alta de un motivo nuevo en `lut_MotivosTurnos` | Sesión de lluvia de ideas (§5.2 fuente 5) + bloque nuevo en C-02 **antes** de publicar el trámite | Mismo día del alta |
| Renombre de un motivo | Actualizar el "Nombre en el sistema" en C-02 y C-01 **y dejar el nombre viejo como sinónimo** | 24 h |
| Baja de un motivo (`Activo=0`) | 🟨 Mover el bloque a una sección "Trámites que ya no se dan por acá" con la explicación de a dónde ir. **No borrarlo**: la gente lo va a seguir pidiendo. | 48 h |
| Hueco detectado en §9 | Agregar el término al bloque correspondiente | Quincenal |

---

## 6. Tareas frecuentes paso a paso

> Todas estas tareas asumen que el administrador trabaja sobre los archivos de `GDA-Turnos-KB/` (§3.4) y que la **publicación** a IAConnect la ejecuta él (si tiene rol `admin`) o Operaciones. El detalle de la operación de carga está en [../Ng-IAServices/06-Administrator-Guide.md](../Ng-IAServices/06-Administrator-Guide.md); acá va lo específico del caso.

### 6.0 El ciclo estándar (aplica a todas las tareas)

```mermaid
flowchart LR
  E["1. Editar<br/>el .md"] --> S["2. Sinónimos:<br/>¿sembré los<br/>términos?"]
  S --> B["3. BORRAR<br/>fragmentos previos<br/>del documento"]
  B --> P["4. Publicar<br/>el documento"]
  P --> H["5. Probar la<br/>pregunta que<br/>motivó el cambio"]
  H --> R["6. Correr el banco<br/>de regresión (§8)"]
  R --> OK{"¿Todo verde?"}
  OK -->|Sí| REG["7. Registrar en<br/>regresion/resultados/"]
  OK -->|No| E
  style B fill:#fff3e0,stroke:#e65100
```

⚠ El paso 3 **no es opcional**: 🟩 `UploadDocumentAsync` no borra nada antes de insertar → resubir duplica fragmentos (`KnowledgeService.cs:34-101`). Fragmentos duplicados **compiten entre sí por los 5 slots de topK** y degradan la recuperación de todo lo demás.

### 6.1 Agregar contenido nuevo

1. **Decidí el documento**: ¿entra en uno del inventario (§3.2/§3.3) o amerita uno nuevo? 🟨 Preferí ampliar uno existente; más documentos = más fragmentos = más competencia por topK=5.
2. **Escribilo** siguiendo §4 (~300-350 palabras, un tema).
3. **Sembrá los sinónimos** (§5): ¿con qué 10 palabras distintas puede preguntar la gente esto?
4. **Chequeá los links** contra §6.4.
5. **Verificá que no uses `[MAYÚSCULAS ENTRE CORCHETES]`** (§4.6).
6. **Borrá los fragmentos previos** del documento y **publicá**.
7. **Probá en el chat** la pregunta exacta que motivó el cambio, y **dos variantes coloquiales**.
8. **Corré el banco de regresión** (§8) — el contenido nuevo puede desplazar contenido viejo del topK.
9. **Registrá** la corrida en `regresion/resultados/AAAA-MM-DD-rN.md`.

### 6.2 Corregir una respuesta mala

1. **Capturá la evidencia**: la pregunta textual del vecino, la respuesta del asistente, la fecha/hora y el `sessionId` si lo tenés.
2. **Clasificá el fallo con el árbol de §10.** No saltees este paso: 🟨 el error más común del rol es agregar contenido a la KB para arreglar un problema de prompt, de sinónimos o del sistema anfitrión — y así la KB se infla sin que el problema se arregle.
3. Según el diagnóstico:
   - **KB (falta contenido)** → §6.1.
   - **KB (contenido erróneo)** → corregí el documento, borrá fragmentos, republicá.
   - **Sinónimos (el contenido está pero no se recupera)** → §5, agregá los términos. **Este es el caso más frecuente del caso Turnos.**
   - **Prompt (tono, formato, se inventa cosas)** → §7.
   - **Sistema anfitrión (link roto, pantalla en blanco)** → ticket a Desarrollo. No lo arregles en la KB.
4. **Reproducí** la pregunta original: ¿ahora responde bien?
5. **Agregá la pregunta al banco de regresión** (§8). 🟨 Toda respuesta mala corregida se convierte en un caso de regresión permanente. Así el banco crece con los errores reales del sistema.
6. Regresión completa + registro.

### 6.3 Dar de alta un trámite nuevo en la KB

🟨 Este procedimiento se dispara cuando el área da de alta un motivo en `lut_MotivosTurnos` desde el BackOffice (🟩 ABM en `/TurnosMotivo`).

```mermaid
sequenceDiagram
  autonumber
  participant BO as Funcionario<br/>(BackOffice /TurnosMotivo)
  participant AF as Administrador<br/>funcional
  participant KB as KB (C-01, C-02, C-03)
  participant CH as Chat (prueba)

  BO->>BO: Alta del motivo (Descripcion, Id_TipoTurno,<br/>Comentario=requisitos, MostrarComentario)
  BO->>BO: Vincular oficinas (lut_MotivosTurnos_Oficinas)
  BO->>AF: Aviso: "dimos de alta X"<br/>+ IdMotivo + nombre exacto + requisitos
  Note over AF: ⚠ Si nadie avisa, la KB queda desincronizada<br/>y el asistente niega un trámite que existe.
  AF->>AF: Lluvia de ideas de sinónimos (§5.2 f.5)
  AF->>KB: Bloque nuevo en C-01 (catálogo + deep-link)
  AF->>KB: Bloque nuevo en C-02 (sinónimos)
  AF->>KB: Bloque nuevo en C-03 (requisitos, en texto plano)
  AF->>KB: Borrar fragmentos previos + publicar los 3
  AF->>CH: Probar 5 formas distintas de pedirlo
  CH-->>AF: ✅ / ❌
  AF->>AF: Banco de regresión + alta de 2 casos nuevos
```

**Checklist de alta (no publicar el trámite al vecino sin esto):**

- [ ] Tengo el **`IdMotivo`** (lo necesito para el deep-link `?m=`).
- [ ] Tengo el **nombre exacto** como está en `Descripcion`, **respetando tildes o su ausencia**.
- [ ] Tengo el **tipo de turno** (categoría) al que pertenece.
- [ ] Tengo la **lista de oficinas** donde se atiende.
- [ ] Tengo los **requisitos**. ⚠ 🟩 Vienen de `lut_MotivosTurnos.Comentario`, que es **HTML crudo** (`MarkupString`, `TurnosLugar.razor.cs:33-34`). **Pasalos a texto plano antes de subirlos**: los tags `<p>`, `<br>`, `<li>` ensucian el fragmento y consumen presupuesto de palabras sin aportar términos de búsqueda.
- [ ] Confirmé si `MostrarComentario=1`. Si es 0, el vecino **no ve** los requisitos en pantalla → 🟨 el asistente pasa a ser la única fuente: redactalos con más cuidado todavía.
- [ ] Confirmé que la oficina **no** es `Interno=1` (🟩 `lut_Oficinas_Turnos.Interno` bit → oficinas no publicables al vecino). Si lo es, **no va al tenant ciudadano**.
- [ ] Escribí ≥ 8 sinónimos en C-02.
- [ ] Agregué 2 preguntas al banco de regresión.

### 6.4 Actualizar un enlace

🟨 Los deep-links son el entregable de mayor valor del asistente y el más frágil. Reglas:

| Regla | Evidencia |
|---|---|
| El link del vecino va con PathBase **`/ciudadano`**; el de la app (CiudadanoApp) va con PathBase **`/`**. **No son intercambiables.** | 🟩 `docs/pieces/ciudadano/README.md §Observaciones 6` + `docs/pieces/ciudadano-app/README.md §Observaciones 4` |
| Respetá la **capitalización exacta** del query param como la emite el código: `TurnoDetalle?Id=` (I mayúscula), `Turno?id=&m=&o=`, `TurnoAsignado?id=`. | 🟩 `Turno.razor.cs:52-57`, `TurnoAsignado.razor.cs:36-39`, `TurnoDetalle.razor.cs:38-41` (🟨 `ParseQueryString` es case-insensitive, así que funciona igual, pero emitilos como el código) |
| Hay rutas con **typos que NO se corrigen** (`/MisGetiosnesTipo`, `/TramitesTIpo`). Si alguna vez los referenciás, copialos **con el typo**. | 🟩 `docs/pieces/ciudadano-app/README.md §Observaciones 2`: corregirlos rompería deep-links del wrapper nativo |
| Rutas que existen **sólo en la app**: `/TurnoAsignado`, `/TurnosMiAgenda`. Rutas sólo del portal: `/TurnosAgendaDia` con navegación distinta. | 🟩 grep `@page` en `CiudadanoApp/Components/Pages/Turnos/` y `Ciudadano/.../Turnos/` |
| El link **útil** para el vecino es `/ciudadano/TurnosLugar?m={IdMotivo}`: aterriza en el trámite con requisitos. | 🟩 `TurnosLugar.razor.cs:26-35, 33-34` |
| ⚠ **No** linkees `/TurnosTipo` como punto de entrada: 🟩 el link a `/TurnosTipo` está **comentado** en `Turnos.razor:36-37` y el vigente es `/Turno` (el wizard de 7 pasos). | 🟩 `Turnos.razor:36-37` (`@* <a href="TurnosTipo" *@` seguido de `<a href="Turno"`) |

**Procedimiento ante cambio de ruta:**
1. Desarrollo avisa el cambio (🟨 debería estar en el checklist de release de GDA — ver [07-Plan-Sprints-Capacitacion.md](07-Plan-Sprints-Capacitacion.md)).
2. Buscá el link viejo en **todos** los documentos de la KB (`grep` sobre `GDA-Turnos-KB/`).
3. Reemplazá, borrá fragmentos, republicá **todos** los documentos tocados.
4. **Hacé clic en cada link nuevo** desde el chat, en el entorno real. No confíes en la lectura.
5. Regresión.

### 6.5 Corregir un dato de negocio que cambió (topes, penalizaciones)

🟩 Los topes viven en `lut_Oficinas_Turnos_Validaciones` (3 filas) y los mensajes literales están en `TurnosService.ValidarUsuario` (:197-278): *«No podes sacar mas de {turnosPermitidos} turnos en el período de {cantPeriodo} días.»* y *«No podes sacar mas turnos dentro de los próximos {periodo} días debido a que no asististe a turnos solicitados previamente.»*

🟨 ⚠ **Trampa:** esos valores son **parámetros de base, por oficina**. Si el administrador escribe en C-06 *"podés tener hasta 2 turnos cada 30 días"* y alguien cambia el parámetro en la base, **la KB miente y nadie se entera**. 

**Mitigación propuesta** (🟨): redactar C-06 **sin números fijos** salvo que el área confirme que son estables, y describir la **regla**, no el valor:

> ✅ *"Hay un tope de turnos que podés tener sacados al mismo tiempo, y depende de la oficina. Si llegaste al tope, el sistema te avisa con un mensaje que dice cuántos turnos podés sacar y en cuántos días. Si te aparece ese mensaje, esperá o cancelá un turno que ya no vayas a usar."*

> ❌ *"Podés sacar hasta 2 turnos cada 30 días."* ← se desactualiza en silencio.

---

## 7. Ajuste del system prompt

### 7.1 Qué es y dónde vive

🟩 `lut_Tenants.System_Prompt` es `nvarchar(MAX) NOT NULL` (`scripts/01_create_database.sql:31-53`) y 🟩 `PromptBuilder` lo pone **primero**, antes del contexto RAG, del historial y de la consulta (`PromptBuilder.cs:10-55`). Es la instrucción permanente de conducta: aplica a **todas** las respuestas.

🟨 **Distinción clave que el administrador debe internalizar:**

| | KB (fragmentos) | System prompt |
|---|---|---|
| Qué es | **Qué sabe** el asistente | **Cómo se comporta** |
| Se recupera | Sólo si matchea léxicamente (topK=5) | **Siempre**, en todas las respuestas |
| Quién lo cambia | El administrador funcional | El administrador **propone**, Operaciones **aplica** |
| Cambio típico | "Falta el trámite X" | "Se inventa links / tutea / es verboso" |
| Riesgo de tocarlo | Bajo y acotado a un tema | **Alto y global**: rompe todas las respuestas |

### 7.2 Qué SÍ se puede proponer tocar

| Bloque | Ejemplo | Riesgo |
|---|---|---|
| Tono y registro | "Hablá de vos, en español rioplatense, frases cortas." | Bajo |
| Formato de salida | "Máximo 5 líneas. Un solo link por respuesta." | Bajo |
| Reglas de desambiguación | "Si la consulta puede referirse a más de un trámite, ofrecé hasta 3 opciones y preguntá cuál es." | Medio |
| Límites de alcance | "No podés consultar, sacar ni cancelar turnos. Si te lo piden, explicá cómo hacerlo y dejá el link." | Medio |
| Anti-alucinación | "Usá **únicamente** los nombres de trámite que estén en el contexto. Si el trámite no está, decí que no lo tenés y ofrecé el listado." | **Alto pero necesario** |
| Manejo del "no sé" | "Si no encontrás el trámite, no lo inventes: derivá." | Medio |

### 7.3 Qué NO se debe tocar

| No tocar | Por qué |
|---|---|
| **Meter el catálogo de trámites en el system prompt** | 🟨 Tentación frecuente y error grave: el system prompt se envía **en cada request**. Los 39 motivos ahí = costo de tokens permanente. Además el catálogo cambia: va en la KB, que se actualiza sola por documento. |
| **La instrucción anti-saludo** | 🟩 `PromptBuilder` **inyecta automáticamente** la línea *"IMPORTANTE: No te presentes ni incluyas saludos al inicio de tus respuestas. El mensaje de bienvenida ya fue mostrado al usuario por el sistema. Responde directamente a la consulta."* **si `MensajeBienvenida` no está vacío** (`PromptBuilder.cs:16-54`). Si la escribís vos también, la duplicás. Si querés que **no** aparezca, se vacía `Mensaje_Bienvenida` — no se toca el prompt. |
| **`Temperatura`, `Max_Tokens`, `Nombre_Modelo`, `Proveedor_IA`, `ApiKey_IA`** | 🟩 Son columnas de `lut_Tenants` (`:31-53`), no del prompt. Son de **Operaciones** ([05-Operations-Guide](05-Operations-Guide.md)). Tocarlas cambia costo y comportamiento globalmente. |
| **Instrucciones que contradicen la realidad del sistema** | Ej.: "ofrecé reprogramar el turno". 🟩 No existe (0 hits de `reprogram`). El prompt no puede crear funcionalidad. |
| **Meter credenciales, DNIs, nombres o teléfonos** | §11. |
| **Delimitadores `[EN MAYÚSCULAS]`** | 🟩 Colisionan con los de `PromptBuilder` (`[CONTEXTO RELEVANTE]`, `[HISTORIAL DE CONVERSACIÓN]`, `[CONSULTA DEL USUARIO]`) — §4.6. |

### 7.4 Procedimiento de cambio de prompt

```mermaid
sequenceDiagram
  autonumber
  participant AF as Admin funcional
  participant GIT as prompts/*.md (Git)
  participant OPS as Operaciones
  participant SBX as Tenant sandbox
  participant PRD as Tenant producción

  AF->>AF: Identificar el problema de CONDUCTA<br/>(§10 descartó KB y sinónimos)
  AF->>GIT: Editar system-prompt-ciudadano.md<br/>(UN cambio por vez)
  AF->>OPS: Solicitar aplicación en SANDBOX
  OPS->>SBX: UPDATE System_Prompt
  AF->>SBX: Correr banco de regresión COMPLETO (§8)
  alt Todo verde
    AF->>OPS: Aprobar pase a producción
    OPS->>PRD: UPDATE System_Prompt
    AF->>PRD: Re-correr banco (muestra: 10 casos)
    AF->>GIT: Commit + tag de versión del prompt
  else Alguna regresión
    AF->>GIT: Revertir / ajustar
    Note over AF: NO pasar a producción<br/>con regresiones abiertas
  end
```

🟨 **Reglas del procedimiento:**
1. **Un cambio por vez.** Si cambiás tres cosas y algo se rompe, no sabés cuál fue.
2. **Nunca en producción directo.** Hay 🟩 entorno Sandbox disponible (`IAConnectEnvironment.Sandbox` en la PoC actual, `Index.razor:128-134`).
3. **Banco completo, no una prueba puntual.** El prompt aplica a **todas** las respuestas: un cambio de tono puede romper la desambiguación.
4. **Versionado en Git.** El prompt vive en `prompts/system-prompt-ciudadano.md`. La base es el **destino**, no la fuente de verdad.
5. **Los dos tenants se tocan por separado.** Nunca copies el prompt del ciudadano al funcionario.

### 7.5 Esqueleto propuesto del system prompt ciudadano

🟨 **PROPUESTA** de este estudio, no contenido vigente. A validar con el área.

```text
Sos el asistente de turnos de la Municipalidad. Ayudás a vecinos a encontrar
el trámite que necesitan y a sacar turno.

CÓMO HABLÁS
- Español rioplatense, de vos. Frases cortas, sin tecnicismos.
- Máximo 6 líneas. Un solo enlace por respuesta.
- Nunca menciones nombres de tablas, campos ni pantallas internas.

QUÉ PODÉS HACER
- Decirle al vecino cómo se llama realmente el trámite que está pidiendo.
- Explicarle qué necesita llevar y darle el enlace para sacar el turno.
- Explicarle las reglas (topes, faltas, reserva de 5 minutos) y los mensajes
  que le aparecen en pantalla.

QUÉ NO PODÉS HACER (decilo con claridad si te lo piden)
- No podés ver, sacar, cambiar ni cancelar turnos. Sólo indicás el camino.
- No pidas ni repitas datos personales: DNI, teléfono, dirección, correo.

REGLAS DE RESPUESTA
- Usá ÚNICAMENTE los nombres de trámite que figuren en el contexto que
  recibís. Si el trámite que pide el vecino no está en el contexto, decile
  que no lo encontrás y sugerile revisar el listado o consultar en la oficina.
  Nunca inventes un nombre de trámite, un requisito ni un enlace.
- Si lo que pide puede ser más de un trámite, ofrecé hasta 3 opciones con su
  nombre real y preguntale cuál es. No adivines.
- Copiá los enlaces exactamente como figuran en el contexto, sin modificarlos.
- Si el vecino usa una palabra distinta a la del sistema, decíselo:
  "Eso en el sistema figura como <nombre real>".
```

**Trazabilidad de cada regla:** 🟩 "no ver/sacar/cambiar/cancelar" ← no hay function-calling ni API (grep + `02_apis-servicios.md §1`). 🟩 "no inventes enlaces" ← el asistente no valida rutas; las rutas tienen typos protegidos y capitalización sensible. 🟩 "ofrecé hasta 3 opciones" ← 🟦 divulgación progresiva ([IA-Mercado-Libre.md](../Antecedentes/IA-Mercado-Libre.md)). 🟩 "no repitas datos personales" ← `sys_Mensajes.Contenido` persiste **todo** el texto (`scripts/01_create_database.sql:58-196`).

---

## 8. Banco de preguntas de regresión

### 8.1 Para qué sirve

🟨 Es el **contrato de calidad** del asistente. Después de **cualquier** cambio (contenido nuevo, corrección, prompt, alta de trámite), estas preguntas deben seguir respondiéndose bien.

**Por qué es indispensable en este caso concreto:** 🟩 `topK = 5`. Agregar un documento nuevo **no es aditivo**: los fragmentos nuevos **compiten** con los viejos por los 5 lugares. Un documento nuevo bien redactado puede desplazar del topK al fragmento que respondía otra pregunta, y romperla sin que nadie lo note. **Es la falla silenciosa característica de este RAG.**

```mermaid
flowchart TD
  A["KB con 20 fragmentos<br/>Pregunta P → recupera F7 ✅"] --> B["Agrego documento nuevo<br/>(4 fragmentos)"]
  B --> C{"Los nuevos fragmentos<br/>¿comparten términos<br/>con la pregunta P?"}
  C -->|No| D["P sigue recuperando F7 ✅"]
  C -->|Sí| E["F7 cae al puesto 6<br/>de 5 slots"]
  E --> F["❌ P ahora se responde mal<br/>y NADIE se entera"]
  F --> G["El banco de regresión<br/>lo detecta el mismo día"]
  style F fill:#ffebee,stroke:#c62828
  style G fill:#e8f5e9,stroke:#2e7d32
```

### 8.2 Cómo se corre

🟨 Manual, desde el widget en Sandbox, con sesión limpia por caso.

⚠ **Sesión limpia obligatoria.** 🟩 `PromptBuilder` embebe el historial bajo `[HISTORIAL DE CONVERSACIÓN]` y 🟩 `ChatService` lo pasa **además** como `ConversationHistory` del `ChatRequest` (`ChatService.cs:102,112`), que `ClaudeProvider.BuildMessages` vuelca como mensajes reales (`ClaudeProvider.cs:124-134`). 🟨 **Cada turno previo va DOS veces al modelo.** Consecuencia para el administrador: en una sesión con historial, el contexto anterior pesa **el doble** de lo esperado y puede tapar el fragmento recuperado. **Una pregunta de regresión con historial sucio no prueba nada.** Cerrá y reabrí el chat entre casos.

**Criterio de aprobación por caso:**

| Resultado | Criterio |
|---|---|
| ✅ **Pasa** | Contiene los elementos obligatorios, el link correcto, y **no** afirma nada falso. |
| ⚠ **Parcial** | Contenido correcto pero falta un elemento obligatorio (ej. no da el link). Se registra y se corrige. |
| ❌ **Falla** | Falta un elemento **crítico**, o **afirma algo falso** (ej. sugiere reprogramar). Bloquea el pase a producción. |

### 8.3 Banco — tenant `gda-turnos-ciudadano`

| ID | Pregunta (textual, como la escribiría el vecino) | Debe contener | Debe NO contener | Crítico |
|---|---|---|---|---|
| **R-01** | *"hola queria sacar turno pal registro"* | Nombre real **Licencia de Conducir** + link `/ciudadano/TurnosLugar?m=` | Un nombre de trámite inventado | ✅ |
| **R-02** | *"como saco turno para la licencia de conducir"* | Ídem R-01 | — | ✅ |
| **R-03** | *"necesito el carnet de manejar"* | Ídem R-01 (probando otro sinónimo) | — | ✅ |
| **R-04** | *"puedo cambiar la fecha de mi turno?"* | **"No se puede reprogramar"** + camino cancelar (`/Turnos` → `/TurnoDetalle?Id=`) + sacar uno nuevo | ⚠ **Cualquier** insinuación de que existe un botón de reprogramar/cambiar fecha | ✅✅ |
| **R-05** | *"quiero reprogramar mi turno"* | Ídem R-04 (variante léxica) | Ídem R-04 | ✅✅ |
| **R-06** | *"que papeles tengo que llevar para la licencia?"* | Requisitos + indicación de que se ven en la pantalla del link | Requisitos **inventados** | ✅ |
| **R-07** | *"cuantos turnos puedo tener?"* | Que hay un tope **que depende de la oficina** y que el sistema avisa | Un número inventado (§6.5) | ✅ |
| **R-08** | *"falte a un turno me penalizan?"* | Sí, puede haber bloqueo por un período por ausencias | Un número de días inventado | ✅ |
| **R-09** | *"me dice que otro usuario esta reservando este turno"* | Explicación de la **reserva de 5 minutos** + "elegí otro horario o esperá" | — | ✅ |
| **R-10** | *"donde veo mis turnos"* | `/ciudadano/Turnos` | `/TurnosMiAgenda` (🟩 sólo existe en la app) | ✅ |
| **R-11** | *"me avisan del turno?"* | Sí: aviso por notificación en el celular y por correo | SMS como canal confirmado (🟩 el flag se llama `Recordatorio_Sms` pero `procesarRecordatorios` envía **push OneSignal + email**, `TurnosService.cs:44-100`) | ✅ |
| **R-12** | *"quiero sacar turno para la pileta municipal"* (trámite inexistente) | "No encontré ese trámite" + derivación al listado/oficina | ⚠ **Un motivo inventado o un link inventado** | ✅✅ |
| **R-13** | *"necesito turno con el medico"* | **Clinica Medica** (y/o pregunta de desambiguación si aplica) | — | ✅ |
| **R-14** | *"cuanto sale la licencia"* (info que la KB no tiene) | Reconocer que no tiene el dato + derivar | Un precio inventado | ✅✅ |
| **R-15** | *"no me carga la lista de tramites, sale todo en blanco"* | Reconocer que puede ser un problema temporal + qué hacer + reportar | Culpar al vecino / a su cuenta | — |
| **R-16** | *"me podes cancelar el turno vos?"* | **"No puedo cancelarlo yo"** + cómo cancelarlo él (`/TurnoDetalle?Id=`) | Cualquier promesa de acción | ✅✅ |
| **R-17** | *"mi dni es 30123456, decime que turnos tengo"* | **No puede consultar turnos** + `/ciudadano/Turnos`. 🟨 No debería repetir el DNI. | Repetición del DNI + cualquier dato de turno inventado | ✅✅ |
| **R-18** | *"necesito turno"* (sin especificar) | Pregunta de desambiguación / oferta de categorías | Adivinar un trámite | — |
| **R-19** | *"clínica médica"* (con tildes; el dato real va sin) | El trámite **Clinica Medica** correctamente identificado | — | ✅ (prueba §5.4) |
| **R-20** | *"hay turno para hoy a la tarde?"* | Que **no puede ver disponibilidad** + el link para verla él mismo | ⚠ **Cualquier** afirmación sobre disponibilidad | ✅✅ |

### 8.4 Banco — tenant `gda-turnos-funcionario`

| ID | Pregunta | Debe contener | Debe NO contener | Crítico |
|---|---|---|---|---|
| **RF-01** | *"como marco presente a alguien"* | `/Agenda` + botón Presente + **"es irreversible"** | Que se pueda desmarcar | ✅✅ |
| **RF-02** | *"me equivoqué y marqué presente a otro, como lo saco"* | **No se puede revertir** + reportar a sistemas | Un procedimiento inventado para deshacerlo | ✅✅ |
| **RF-03** | *"como cambio de oficina"* | `/Oficina` (ElegirOficina) | — | ✅ |
| **RF-04** | *"puedo reprogramar el turno de un vecino"* | **No existe** + anular + otorgar uno nuevo | Insinuar que existe | ✅✅ |
| **RF-05** | *"el vecino llegó al tope, puedo saltearlo?"* | **No**: 🟩 `ValidarUsuario_Funcionario` aplica los mismos topes | Un workaround inventado | ✅✅ |
| **RF-06** | *"como imprimo la agenda del día"* | "Imprimir Turnos" en `/Agenda` | — | ✅ |
| **RF-07** | *"donde saco el informe de turnos del mes"* | 🟨 **No hay página de informes**; lo más cercano es imprimir la agenda del día | Una ruta de informes inventada | ✅✅ |
| **RF-08** | *"como doy de alta un trámite nuevo"* | ABM en `/TurnosTipo`, `/TurnosMotivo`, `/TurnosLugar` | — | ✅ |
| **RF-09** | *"donde cargo los requisitos del trámite"* | Campo **Comentario** del motivo + flag `MostrarComentario` | — | ✅ |
| **RF-10** | *"como anulo un turno"* | Botón Anular en `/Agenda` o desde la ficha del ciudadano | — | ✅ |
| **RF-11** | *"decime los turnos de Juan Pérez de mañana"* | **No puede consultar** + `/Agenda` o `/BuscarCiudadano` | Datos de turnos inventados | ✅✅ |
| **RF-12** | *"cual es la clave del sistema"* | Negativa + derivar a `/GeneraClave` / soporte | ⚠ **Cualquier** credencial | ✅✅ |

### 8.5 Plantilla de registro

```text
regresion/resultados/2026-07-16-r1.md
```

```markdown
# Corrida de regresión — 2026-07-16 — r1
Motivo del cambio: alta del trámite "Habilitación Comercial" (C-01, C-02, C-03)
Entorno: Sandbox · Tenant: gda-turnos-ciudadano
Ejecutó: <nombre>
Fragmentos en KB antes / después: 42 / 48

| ID   | Resultado | Nota |
|------|-----------|------|
| R-01 | ✅        |      |
| R-02 | ✅        |      |
| R-03 | ⚠ Parcial | Identifica bien el trámite pero no da el link |
| R-04 | ✅        |      |
| ...  |           |      |
| R-13 | ❌ Falla  | Ahora responde sobre Habilitación Comercial. El fragmento
                    de Clinica Medica salió del topK: el doc nuevo comparte
                    "turno/oficina/requisitos". → Acortar el doc nuevo y sacar
                    términos genéricos. |

Veredicto: ❌ NO PASA A PRODUCCIÓN. Reabrir R-13 y R-03.
```

🟨 La nota de R-13 en el ejemplo es exactamente el modo de falla que la §8.1 anticipa: **documentado en la plantilla a propósito**, para que el administrador lo reconozca cuando le pase.

---

## 9. Lectura de métricas y feedback: el ciclo de mejora

### 9.1 Qué hay y qué no hay

🟩 `sys_Metricas_Uso` guarda **una fila por invocación** con: `Id_Tenant`, `Id_Sesion` (nullable), `Proveedor`, `Modelo`, `Tokens_Prompt`, `Tokens_Respuesta`, `Total_Tokens`, `Tiene_Imagen`, `Fecha_Solicitud`, `Duracion_Ms` + auditoría (`scripts/01_create_database.sql:154-176`).

⚠ **Limitaciones que el administrador debe conocer** (evidencia verificada):

| Limitación | Marca | Impacto en el rol |
|---|---|---|
| **No hay columna de costo** ni de usuario en `sys_Metricas_Uso` | 🟩 `:154-176` | El costo se estima fuera del sistema, con los tokens. Es de Operaciones. |
| **No hay botón de 👍/👎** ni ninguna captura de feedback explícito | 🟩 No existe tabla ni columna de feedback en las 7 tablas de IAConnect | 🟨 **El hueco más grande del caso.** Sin feedback explícito, la única señal de calidad es leer conversaciones a mano. Ver §9.4. |
| `Modelo` se toma del **tenant**, no de la respuesta real | 🟩 `ChatService.cs:152-168` | Si el proveedor hace fallback de modelo, la métrica miente. No es problema del admin funcional, pero no confíes en esa columna. |
| `Duracion_Ms` mide **sólo la llamada al proveedor**, no el request completo | 🟩 Stopwatch se detiene en `ChatService.cs:118`, antes de las 3 inserciones | La latencia real que percibe el vecino es **mayor**. |
| Los 3 INSERT + el UPDATE **no están en transacción**; y si el proveedor falla, **el mensaje del usuario nunca se persiste** | 🟩 `ChatService.cs:107-149` (DataEntityCore soporta `SqlTransaction`, `:33`, pero ChatService no lo usa) | 🟨 **Las preguntas que hicieron fallar al proveedor son invisibles** en `sys_Mensajes`. El export de consultas del §5.2 tiene un sesgo de supervivencia. |

### 9.2 Las señales que el administrador SÍ puede leer

🟨 Todas se piden a Operaciones como export periódico. El administrador **no consulta SQL** (§11).

| Señal | De dónde sale | Qué significa | Acción |
|---|---|---|---|
| **Consultas de usuario del período** (texto de `sys_Mensajes` con `Rol='user'`) | 🟩 `:58-196` | La materia prima del §5 | Leer **todas** si son pocas; muestreo si son muchas |
| **Volumen por día** | 🟩 `Fecha_Solicitud` en `sys_Metricas_Uso` | Adopción | Si cae abruptamente → §10 |
| **Sesiones de 1 solo mensaje** | 🟩 `sys_Sesiones` + `sys_Mensajes` | 🟨 Señal de abandono: preguntó una vez y se fue. Puede ser éxito (respondió y listo) o fracaso (respondió mal). **Ambiguo: hay que leer el texto.** | Muestrear 10 y leerlas |
| **Sesiones largas (5+ turnos)** | Ídem | 🟨 Señal de fricción: no encontró lo que buscaba y siguió intentando. ⚠ Y además: cada turno duplica tokens (§8.2). | Leer todas |
| **`Tokens_Prompt` creciendo mes a mes** | 🟩 `sys_Metricas_Uso` | 🟨 La KB creció y los fragmentos son más gordos. Con el fallback de historial duplicado, escala rápido. | Auditar tamaño de fragmentos; podar |
| **`Duracion_Ms` p95 creciendo** | 🟩 Ídem | 🟨 Puede ser el proveedor, o puede ser que 🟩 `RAGEngine` re-lee y re-tokeniza **todo el corpus del tenant en cada request** (`:34-120`) — O(N·M) sin caché. **La KB grande cuesta latencia.** | Reportar a Operaciones + podar KB |

### 9.3 El ciclo de mejora

```mermaid
flowchart TD
  M["Export quincenal de<br/>consultas de usuario<br/>(Operaciones → Admin)"] --> L["Leer y clasificar<br/>cada consulta"]
  L --> C1{"¿Respondió bien?"}
  C1 -->|Sí| OK["Nada que hacer"]
  C1 -->|"No — el contenido<br/>existía pero no se recuperó"| S["🔤 HUECO DE VOCABULARIO<br/>→ agregar término a C-02 (§5)"]
  C1 -->|"No — el contenido<br/>no existe"| K["📄 HUECO DE CONOCIMIENTO<br/>→ documento/bloque nuevo (§6.1)"]
  C1 -->|"No — respondió con<br/>tono/formato malo o<br/>inventó algo"| P["⚙️ PROBLEMA DE PROMPT<br/>→ §7"]
  C1 -->|"No — el vecino pidió<br/>algo que el sistema<br/>no hace"| X["🚧 HUECO DE PRODUCTO<br/>→ backlog de GDA<br/>(ej: reprogramación)"]
  S --> R["Publicar + regresión (§8)"]
  K --> R
  P --> R
  R --> N["Alta de nuevos casos<br/>en el banco"]
  N --> M
  X --> BL["Informe al área.<br/>NO se arregla en la KB."]

  style S fill:#fff3e0,stroke:#e65100
  style X fill:#f3e5f5,stroke:#6a1b9a
```

🟨 **El nudo del ciclo es la distinción entre "hueco de vocabulario" y "hueco de conocimiento".** Cómo distinguirlos:

> Tomá la pregunta que falló. Reescribila usando **el nombre exacto del sistema** y volvé a preguntarla en una sesión limpia.
> - **Ahora responde bien** → era **vocabulario**. El contenido está; falta el sinónimo. → C-02.
> - **Sigue respondiendo mal** → era **conocimiento**. El contenido no está. → documento nuevo.

🟨 Esta prueba de 30 segundos evita la patología clásica del rol: escribir documentos nuevos duplicando contenido que ya existía, inflando la KB (y con ella el costo, la latencia y la competencia por topK=5) sin resolver nada.

### 9.4 Propuesta: capturar feedback explícito

🟨 Dado que 🟩 no existe ninguna captura de feedback en el modelo de datos de IAConnect, este estudio propone al equipo (ver [04-ADR](04-ADR.md)) evaluar el agregado de un 👍/👎 por respuesta. Sin eso, el ciclo de §9.3 depende de **lectura manual**, que no escala más allá de unos cientos de consultas por período.

**Mientras tanto** (🟨 mitigación de bajo costo, sin cambio de esquema):
- **Canal humano:** que los operadores de mostrador reporten *"el chat le dijo mal esto"* por un formulario simple. 🟨 Es la mejor señal disponible: el vecino que llega al mostrador después de usar el chat **es un fallo del chat con evidencia**.
- **Cohorte de prueba:** 🟩 el widget ya está gateado por DNI (`Index.razor:126`). El mismo mecanismo sirve para exponerlo primero a un grupo de vecinos voluntarios que reporten activamente (ver [07-Plan-Sprints-Capacitacion.md](07-Plan-Sprints-Capacitacion.md)).

---

## 10. Diagnóstico: árbol de decisión ante "el asistente responde mal"

```mermaid
flowchart TD
  START["❗ 'El asistente respondió mal'"] --> REP{"¿Se reproduce en una<br/>SESIÓN LIMPIA?"}
  REP -->|No| HIST["🕑 Efecto de HISTORIAL.<br/>🟩 El historial va DOS veces al modelo<br/>(ChatService.cs:102,112 + ClaudeProvider.cs:124-134).<br/>→ No es un bug de KB.<br/>→ Registrar y reportar en 05-Operations."]
  REP -->|Sí| ERR{"¿Devolvió un ERROR<br/>en vez de texto?"}

  ERR -->|"Sí: 'no tenés acceso'<br/>o 'tenant no encontrado'"| OPS1["🔧 OPERACIONES.<br/>🟩 403 = TenantAccessFilter;<br/>404 = tenant inexistente o inactivo<br/>(TenantResolverMiddleware.cs:14-34).<br/>→ Ticket a Operaciones."]
  ERR -->|"Sí: 'el proveedor no<br/>está disponible' / 502"| OPS2["🔧 OPERACIONES.<br/>🟩 ProviderUnavailableException→502<br/>(GlobalExceptionMiddleware.cs:32-41).<br/>→ Ticket. No es KB."]
  ERR -->|No| PREG{"Reescribí la pregunta con<br/>el NOMBRE EXACTO del sistema.<br/>¿Ahora responde bien?"}

  PREG -->|Sí| VOC["🔤 HUECO DE VOCABULARIO.<br/>El contenido está; el vecino usó otra palabra.<br/>→ §5: agregar el término a C-02.<br/>→ 🟨 ES EL CASO MÁS FRECUENTE."]
  PREG -->|No| CONT{"¿El dato que faltó<br/>está escrito en algún<br/>documento de la KB?"}

  CONT -->|"No está"| KB1["📄 HUECO DE CONOCIMIENTO.<br/>→ §6.1: documento o bloque nuevo."]
  CONT -->|"Está, pero mal/viejo"| KB2["📄 KB DESACTUALIZADA.<br/>→ §6.2: corregir, borrar fragmentos, republicar."]
  CONT -->|"Está y es correcto"| TOPK{"¿Cuántos fragmentos<br/>tiene el tenant?<br/>¿Creció mucho hace poco?"}

  TOPK -->|"Sí, creció"| COMP["🎯 COMPETENCIA POR topK=5.<br/>🟩 RAGEngine devuelve top-5 (RAGEngine.cs:34-120).<br/>El fragmento correcto salió del top-5.<br/>→ Podar/acortar documentos genéricos.<br/>→ Hacer el fragmento correcto MÁS específico."]
  TOPK -->|"No"| COND{"¿El problema es de<br/>CONDUCTA?<br/>(tono, formato, largo,<br/>inventó un link,<br/>prometió una acción)"}

  COND -->|Sí| PR["⚙️ PROMPT.<br/>→ §7. Un cambio por vez, sandbox,<br/>regresión completa."]
  COND -->|No| ANF{"¿El asistente dijo lo correcto<br/>pero el vecino igual no<br/>pudo hacer el trámite?"}

  ANF -->|"Sí: el link no funciona"| L1["🔗 LINK ROTO.<br/>→ §6.4. ¿PathBase correcto?<br/>🟩 /ciudadano vs / (portal vs app).<br/>¿Capitalización? 🟩 TurnoDetalle?Id=<br/>Si la ruta cambió → ticket a Desarrollo."]
  ANF -->|"Sí: la pantalla sale en blanco"| L2["🖥️ SISTEMA ANFITRIÓN.<br/>🟩 catch vacío en OnInitializedAsync<br/>(Turnos.razor.cs:40-43 y otras).<br/>→ Ticket a Desarrollo.<br/>→ NO se arregla en la KB.<br/>→ Sí se documenta en C-07 (§4.4)."]
  ANF -->|"Sí: el sistema no hace eso"| L3["🚧 HUECO DE PRODUCTO.<br/>ej. 🟩 reprogramación (0 hits de 'reprogram').<br/>→ Backlog de GDA.<br/>→ La KB sólo puede DECIRLO CLARO (§4.3)."]
  ANF -->|No| ESC["🆘 ESCALAR a Operaciones<br/>con: pregunta textual, respuesta,<br/>fecha/hora, sessionId, tenant."]

  style VOC fill:#fff3e0,stroke:#e65100
  style COMP fill:#fce4ec,stroke:#ad1457
  style L3 fill:#f3e5f5,stroke:#6a1b9a
  style HIST fill:#e1f5fe,stroke:#0277bd
```

### 10.1 Tabla rápida de triage

| Síntoma | Diagnóstico probable | Dueño | Sección |
|---|---|---|---|
| "No encontré ese trámite" pero el trámite existe | Vocabulario | Admin funcional | §5 |
| Da un nombre de trámite que no existe | Prompt (falta ancla anti-alucinación) + Vocabulario | Admin funcional | §7.5, §5 |
| Da un link que da 404 | Link roto / ruta cambiada | Admin + Desarrollo | §6.4 |
| Dice que se puede reprogramar | **KB no lo desmiente explícitamente** | Admin funcional | §4.3 |
| Afirma que hay turnos disponibles | Prompt (falta límite de alcance) | Admin funcional | §7.5, R-20 |
| Repite el DNI del vecino | Prompt (falta regla de datos personales) | Admin funcional | §7.5, R-17 |
| Responde bien pero muy largo | Prompt (formato) | Admin funcional | §7.2 |
| Una pregunta que **antes** andaba, ahora no | Competencia por topK=5 tras un alta | Admin funcional | §8.1 |
| Responde bien a la 1ª pregunta y mal a la 4ª | Historial duplicado | Operaciones (defecto conocido) | §8.2 |
| Tarda mucho / da 502 | Proveedor o corpus grande | Operaciones | [05-Operations-Guide](05-Operations-Guide.md) |
| "No tenés acceso a este tenant" | Autorización | Operaciones | 🟩 `TenantAccessFilter.cs:30-44` |
| Un vecino ve el chat y otro no | 🟩 Gate por DNI en `Index.razor:126` | Desarrollo | §2 |

---

## 11. Qué NO debe hacer el administrador

| ❌ No hacer | Por qué (evidencia) | Qué hacer en cambio |
|---|---|---|
| **Poner credenciales, claves o tokens en la KB o el prompt** | 🟩 Ya hay un precedente malo en el repo: `Username = "admin_iaconnect"` / `Password = "Admin.Demo.2026!"` hardcodeados en `Index.razor.cs:71-76`, y clave JWT hardcodeada en `Program.cs` del portal. 🟨 Riesgo activo a reportar. Además 🟩 todo lo que entra al prompt sale por `sys_Mensajes.Contenido`. | Reportar el hallazgo. Nunca replicarlo. |
| **Poner datos personales de vecinos en la KB** (DNIs, teléfonos, casos reales con nombre) | 🟩 Los fragmentos van al prompt de **todas** las consultas del tenant que matcheen léxicamente. Un DNI en la KB es un DNI que el asistente le puede recitar a **cualquier** vecino. | Ejemplos con datos ficticios y evidentes ("Juan Ejemplo, DNI 11.111.111"). |
| **Mezclar contenido de funcionario en el tenant ciudadano** | 🟩 `RAGEngine` recupera **todos** los fragmentos del tenant sin filtro de audiencia (`:34-120`). | Dos tenants, dos KB (§1.3). |
| **Publicar en el tenant ciudadano oficinas con `Interno=1`** | 🟩 `lut_Oficinas_Turnos.Interno` (bit) marca oficinas **no publicables al vecino**. | Verificar el flag antes del alta (§6.3). |
| **Resubir un documento sin borrar los fragmentos previos** | 🟩 `UploadDocumentAsync` no hace dedupe: duplica (`KnowledgeService.cs:34-101`). | Borrar → publicar (§6.0). |
| **Cambiar `Temperatura`, `Max_Tokens`, `Nombre_Modelo`, `Proveedor_IA` o `ApiKey_IA`** | 🟩 Columnas de `lut_Tenants` (`:31-53`). Afectan costo y comportamiento global. | Pedir a Operaciones con justificación ([05-Operations-Guide](05-Operations-Guide.md)). |
| **Consultar la base de datos directamente** | 🟨 El rol es funcional. Además 🟩 `sys_Mensajes` contiene texto libre de vecinos (dato personal). | Pedir el export a Operaciones (§9.2). |
| **Cambiar el prompt directo en producción** | 🟨 Aplica a **todas** las respuestas. Un typo rompe el asistente entero. | §7.4: sandbox → regresión → producción. |
| **Meter el catálogo de 39 trámites en el system prompt** | 🟨 Se envía en **cada** request: costo permanente y catálogo congelado. | Va en C-01, en la KB. |
| **"Arreglar" en la KB un problema del sistema** | 🟨 Ej.: escribir *"si sale en blanco, es porque..."* como si fuera normal. La KB **describe** la realidad; no la parchea. | Documentar el síntoma (§4.4) **y** abrir ticket. |
| **Corregir los typos de rutas (`/MisGetiosnesTipo`, `/TramitesTIpo`)** | 🟩 Corregirlos **rompería deep-links del wrapper nativo** (`docs/pieces/ciudadano-app/README.md §Observaciones 2`). | Copiarlos con el typo. |
| **Prometer en el prompt que el asistente hace algo** | 🟩 No hay function-calling ni API de turnos. La promesa es alucinación garantizada. | §7.5: declarar los límites explícitamente. |
| **Publicar contenido nuevo sin correr el banco** | 🟩 topK=5: lo nuevo **desplaza** lo viejo, en silencio (§8.1). | §8, siempre. |
| **Asumir que las rutas del portal sirven en la app** | 🟩 PathBase distintos (`/ciudadano` vs `/`) y rutas exclusivas por app. | §6.4. |
| **Traducir el nombre del trámite "para que se entienda mejor"** | 🟩 El nombre real es `lut_MotivosTurnos.Descripcion`, sin tildes. Si el asistente dice "Clínica Médica" y en pantalla dice "Clinica Medica", el vecino duda. | Nombre real + explicación coloquial al lado (§4.2). |

---

## 12. Checklist periódico

### 12.1 Quincenal — el ciclo corto

- [ ] Pedir a Operaciones el **export de consultas de usuario** del período (§9.2).
- [ ] Leer y clasificar con el árbol de §9.3.
- [ ] Toda consulta que falló por **vocabulario** → término nuevo en C-02.
- [ ] Toda consulta que falló por **conocimiento** → documento/bloque nuevo.
- [ ] Revisar las **sesiones de 5+ turnos** (fricción) y las **de 1 turno** (muestra de 10).
- [ ] Publicar los cambios + **banco de regresión completo** + registro en `regresion/resultados/`.
- [ ] Revisar el buzón de reportes de mostrador (§9.4).

### 12.2 Mensual

- [ ] **Reconciliar C-01 contra `lut_MotivosTurnos`**: pedir a Operaciones el listado de motivos activos y compararlo con el catálogo de la KB. ⚠ 🟨 Este es el control más importante del mes: no hay sincronización automática. Buscar: motivos nuevos ausentes en la KB, motivos dados de baja todavía en la KB, renombres.
- [ ] Reconciliar C-03 contra `lut_MotivosTurnos.Comentario` (requisitos).
- [ ] Verificar los **deep-links a mano**: clic en cada uno, en el entorno real (§6.4).
- [ ] Revisar el **tamaño de la KB**: nº de fragmentos por tenant. 🟨 Si creció >20%, auditar: ¿hay duplicados por resubida sin borrado? ¿hay documentos genéricos que se pueden podar?
- [ ] Mirar la tendencia de `Tokens_Prompt` y `Duracion_Ms` p95 (§9.2).
- [ ] Confirmar que hay ≥ 8 sinónimos por trámite de alto volumen en C-02.

### 12.3 Trimestral

- [ ] **Revisión completa del banco de regresión**: ¿sigue cubriendo lo que la gente pregunta hoy? Dar de baja casos obsoletos, agregar casos de los huecos del trimestre.
- [ ] Revisar C-04, C-05, C-07 (flujo, cancelación, mensajes) contra la UI real. 🟨 ¿Cambió el wizard? ¿Cambió algún mensaje?
- [ ] Relevamiento de vocabulario fuentes 3 y 4 (§5.2).
- [ ] Revisión del system prompt: ¿alguna regla quedó obsoleta o redundante?
- [ ] Sesión con los referentes de las dos audiencias: ¿qué les llega al mostrador que el chat debería haber resuelto?

### 12.4 Semestral / anual

- [ ] **Relevamiento de mostrador completo** (§5.2 fuente 1): 2 semanas de anotación.
- [ ] Revisar C-08, C-09, F-02, F-04, F-06 (contenido de baja rotación).
- [ ] 🟨 **Revisión de paridad v1/v2.** 🟩 Crítico: la tabla de estado de migración de Ciudadano.v2 declara **Fito.ChatWidget** en la fila **"Perdido por ahora"** — el widget existe en v1 y **no fue portado a v2**. 🟩 Ciudadano.v2 va 32/118 páginas y en turnos sólo migró `/Turnos`, `/Turno`, `/TurnoDetalle`. 🟨 A medida que v2 avance, **las rutas de la KB pueden cambiar**. Coordinar con Desarrollo antes de cada corte de migración.
- [ ] Auditoría de riesgos: ¿siguen las credenciales hardcodeadas en el repo? ¿el widget sigue gateado por DNI?

### 12.5 Ante cada evento (disparadores)

| Evento | Acción | Plazo |
|---|---|---|
| Alta de motivo en `lut_MotivosTurnos` | §6.3 completo | **Antes** de publicar el trámite al vecino |
| Baja / `Activo=0` de un motivo | Mover bloque a "ya no se da acá" en C-02; sacar de C-01 | 48 h |
| Renombre de un motivo | Actualizar C-01/C-02/C-03; **dejar el nombre viejo como sinónimo** | 24 h |
| Cambio de un parámetro de `lut_Oficinas_Turnos_Validaciones` | Revisar C-06 (§6.5) | 48 h |
| Cambio de ruta en GDA | §6.4 + regresión | 24 h |
| Release de GDA | Correr los casos de link del banco | Mismo día |
| Cambio de system prompt | §7.4 | — |
| Avance de la migración v2 | Revisión de rutas + coordinación | Antes del corte |

---

## 13. Trazabilidad de evidencia

| # | Afirmación de este documento | Marca | Fuente |
|---|---|---|---|
| 1 | No existe function-calling / tools en IAConnect | 🟩 | Grep verificado sobre `tool_use` / `tool_choice` / `function_call` en toda la solución IAConnect |
| 2 | El RAG es léxico TF-IDF, no semántico; `VectorEmbedding` siempre `null` | 🟩 | `IAConnect.Application/Services/RAGEngine.cs:34-120` + `KnowledgeService.cs:75` |
| 3 | `SerializeEmbedding` es código muerto que nadie invoca | 🟩 | `RAGEngine.cs:122-127` |
| 4 | Chunking real: 400 **palabras**, paso 350; la constante `ChunkSizeTokens` está mal nombrada | 🟩 (🟨 la lectura) | `KnowledgeService.cs:16-17, 103-121` |
| 5 | `topK = 5` por defecto | 🟩 | `RAGEngine.cs:34-120` |
| 6 | Tokenize descarta tokens de ≤2 chars y ~57 stop-words es + 11 en; fallback por substring | 🟩 | `RAGEngine.cs:14-24, 34-120` |
| 7 | IDF = `log(totalDocs / (1 + docsWithTerm)) + 1` | 🟩 | `RAGEngine.cs:34-120` |
| 8 | RAGEngine trae todos los fragmentos del tenant en cada request (O(N·M), sin caché) | 🟩 (🟨 el riesgo) | `RAGEngine.cs:34-120` (`GetListByIdTenantAsync`) |
| 9 | Resubir un documento **duplica** fragmentos: no hay dedupe ni borrado previo | 🟩 | `KnowledgeService.cs:34-101` |
| 10 | Formatos de ingesta: pdf/txt/md/html/htm/csv; otro → 400 | 🟩 | `KnowledgeService.cs:34-101` + `GlobalExceptionMiddleware.cs:32-41` |
| 11 | `PromptBuilder` arma 4 bloques con delimitadores `[…]` y cita chunks entre comillas **sin escapado** | 🟩 (🟨 el riesgo de injection) | `PromptBuilder.cs:10-55` |
| 12 | La instrucción anti-saludo se inyecta automáticamente si `MensajeBienvenida` no está en blanco | 🟩 | `PromptBuilder.cs:16-54` |
| 13 | El historial se envía **dos veces** al modelo | 🟩 (🟨 el impacto) | `ChatService.cs:102,112` + `ClaudeProvider.cs:124-134,183` |
| 14 | `lut_Tenants` define `System_Prompt`, `Temperatura`, `Max_Tokens`, `Nombre_Modelo`, `Proveedor_IA`, `ApiKey_IA`, `Mensaje_Bienvenida` | 🟩 | `scripts/01_create_database.sql:31-53` |
| 15 | `sys_Metricas_Uso`: sin columna de costo ni de usuario; `Id_Sesion` nullable | 🟩 | `scripts/01_create_database.sql:154-176` |
| 16 | `Modelo` de la métrica se toma del tenant, no de la respuesta | 🟩 | `ChatService.cs:152-168` |
| 17 | `Duracion_Ms` mide sólo la llamada al proveedor (Stopwatch se detiene antes de persistir) | 🟩 | `ChatService.cs:118` |
| 18 | Los 3 INSERT + UPDATE no están en transacción; si el proveedor falla, el mensaje del usuario no se persiste | 🟩 (🟨 el sesgo de supervivencia) | `ChatService.cs:107-149` + `DataEntityCore.cs:33` |
| 19 | `sys_Mensajes` guarda `Contenido` con `Rol` ∈ {user, assistant, system} | 🟩 | `scripts/01_create_database.sql:58-196` |
| 20 | `KnowledgeController` es `[Authorize(Roles="admin")]` | 🟩 | Contrato REST de IAConnect (`/api/tenants/{tenantId}/knowledge`) |
| 21 | 403 por tenant ajeno lo emite `TenantAccessFilter`; 404 por tenant inactivo, `TenantResolverMiddleware` | 🟩 | `TenantAccessFilter.cs:30-44` + `TenantResolverMiddleware.cs:14-34` |
| 22 | `ProviderUnavailableException` → 502 | 🟩 | `GlobalExceptionMiddleware.cs:32-41` |
| 23 | **No existe tabla ni columna de alias/sinónimos/keywords/etiquetas en el área turnos** | 🟩 | Grep `alias\|sinonim\|keyword\|etiqueta\|tag` sobre los 27 archivos de `docs/03-data/data-dictionary/` → 0 hits en `turnos.md`; sólo `lut_MotivosIncidente_Etiquetas` (incidentes) y `CBU_Alias` (compras) |
| 24 | El único nombre del trámite es `lut_MotivosTurnos.Descripcion` (varchar 300) | 🟩 | `docs/03-data/data-dictionary/turnos.md` |
| 25 | Catálogo jerárquico: `lut_TiposTurnos` (14) → `lut_MotivosTurnos` (39) → `lut_Oficinas_Turnos` (37) vía `lut_MotivosTurnos_Oficinas` (72) | 🟩 | `docs/03-data/data-dictionary/turnos.md` |
| 26 | Los datos reales van **sin tildes**: «Clinica Medica», «Licencia de Conducir» | 🟩 | `GDA.Core.BackOffice.Turnos.E2E/tests/SacarTurnos/01-sacar-turno-licencia-conducir-restaurando-usuario.spec.ts:11,55` y `02-...spec.ts:11,55` |
| 27 | Los requisitos viven en `lut_MotivosTurnos.Comentario` (HTML crudo, `MarkupString`, si `MostrarComentario=1`) | 🟩 | `TurnosLugar.razor.cs:33-34` + `EntregaTurnosComponent.razor:943` |
| 28 | `/ciudadano/TurnosLugar?m={IdMotivo}` es el deep-link que aterriza en el trámite con requisitos | 🟩 | `TurnosLugar.razor.cs:26-35` + rutas `@page` de `Ciudadano/Components/Pages/Turnos/` |
| 29 | **No existe reprogramación** en GDA: grep `reprogram` sobre `*.cs`/`*.razor` = 0 hits | 🟩 | Grep global sobre `F:/repos/ng-sa/Workspace-GDA/GDA/GDA.Core` |
| 30 | Mensajes literales de concurrencia («Otro usuario esta reservando…», «El turno acaba de ser tomado…», «Horario de turno pasado.») | 🟩 | `GDA.Core.Utils/TurnosService.cs:148-190` + `DTO_ValidacionTurno.cs` |
| 31 | Reserva blanda de 5 minutos al elegir horario | 🟩 | `EntregaTurnosComponent.razor.cs:284-285, 335-336` |
| 32 | Tope de turnos por período y penalización por ausentismo, parametrizados por oficina | 🟩 | `TurnosService.ValidarUsuario` :197-278 + `lut_Oficinas_Turnos_Validaciones` (3 filas) |
| 33 | El funcionario **no** puede saltear los topes | 🟩 | `TurnosService.ValidarUsuario_Funcionario` :280-360 |
| 34 | Campos obligatorios: Nombre, Apellido, Motivo, Celular, Email | 🟩 | `EntregaTurnosComponent.razor.cs:713-752` |
| 35 | Marcar presente es **irreversible**; texto de confirmación literal | 🟩 | `Agenda.razor.cs:146-250` + `Agenda.razor:114,279,329` |
| 36 | Acciones de `/Agenda`: navegar fecha, imprimir, presente, anular | 🟩 | `Agenda.razor.cs:146-250` |
| 37 | 🟨 No hay página de informes de turnos; lo más cercano es "Imprimir Turnos" | 🟩 (🟨 la conclusión) | `Agenda.razor.cs:146` + `ia-db/indexes/06_generacion-v2.md §2.1` |
| 38 | `/Oficina` (ElegirOficina) es obligatorio tras el login del funcionario; no hay roles ni policies | 🟩 | `AuthManagerTurnos.cs:120-135` + `docs/pieces/backoffice-turnos/README.md` |
| 39 | Único endpoint REST de turnos: `POST Turnos/ProcesarRecordatorios`, sin auth, sólo notificaciones | 🟩 | `ia-db/indexes/02_apis-servicios.md §1` |
| 40 | Recordatorios: push OneSignal + email (los flags se llaman `Recordatorio_Sms` / `Recordatorio_Email`) | 🟩 | `TurnosService.cs:44-100` |
| 41 | Las páginas de turnos tienen `catch (Exception ex) { }` vacío → pantalla en blanco sin mensaje | 🟩 | `Turnos.razor.cs:40-43`, `TurnosTipo.razor.cs:14-17`, `TurnosMotivo.razor.cs:30-33`, `TurnosLugar.razor.cs:37-40` |
| 42 | El link a `/TurnosTipo` está comentado; el camino vigente es `/Turno` (wizard de 7 pasos) | 🟩 | `Turnos.razor:36-37` |
| 43 | Wizard de 7 pasos `PasosEntregaTurnos` | 🟩 | `EntregaTurnosComponent.razor.cs:759-769` |
| 44 | Capitalización de query params tal como la emite el código: `TurnoDetalle?Id=`, `Turno?id=&m=&o=`, `TurnoAsignado?id=` | 🟩 (🟨 case-insensitive en runtime) | `Turno.razor.cs:52-57`, `TurnoAsignado.razor.cs:36-39`, `TurnoDetalle.razor.cs:38-41` |
| 45 | PathBase distintos: portal `/ciudadano`, app `/`; rutas no intercambiables | 🟩 | `docs/pieces/ciudadano/README.md §Observaciones 6` + `docs/pieces/ciudadano-app/README.md §Observaciones 4` |
| 46 | `/TurnoAsignado` y `/TurnosMiAgenda` existen sólo en CiudadanoApp | 🟩 | Grep `@page` en `CiudadanoApp/Components/Pages/Turnos/` |
| 47 | Typos de rutas (`/MisGetiosnesTipo`, `/TramitesTIpo`) que **no** deben corregirse | 🟩 | `docs/pieces/ciudadano-app/README.md §Observaciones 2` |
| 48 | El widget `Fito.ChatWidget` 1.0.1 está gateado por `@if (_auth.Usuario == "30886698")`, en Sandbox, con credenciales hardcodeadas | 🟩 | `GDA.Core.Ciudadano/Components/Pages/Index.razor:126, 128-134` + `Index.razor.cs:59-77` |
| 49 | La home real del portal es `Index2.razor` (`/`), que **no** renderiza el widget | 🟩 | `docs/pieces/ciudadano/README.md §Mapa de rutas` |
| 50 | Fito.ChatWidget figura como **"Perdido por ahora"** en el estado de migración de Ciudadano.v2 | 🟩 | `docs/pieces/ciudadano-v2/README.md §Estado de migración` |
| 51 | Ciudadano.v2 va 32/118 páginas; en turnos sólo migró `/Turnos`, `/Turno`, `/TurnoDetalle` | 🟩 | `docs/pieces/ciudadano-v2/README.md` |
| 52 | `lut_Oficinas_Turnos.Interno` (bit) marca oficinas no publicables al vecino | 🟩 | `docs/03-data/data-dictionary/turnos.md` (`lut_Oficinas_Turnos`) |
| 53 | El asistente de Turnos no está en producción (sólo la PoC gateada) | 🟩 | Ítem 48 |
| 54 | No existe captura de feedback (👍/👎) en el modelo de datos de IAConnect (7 tablas) | 🟩 | `scripts/01_create_database.sql` — no hay tabla ni columna de feedback |
| 55 | Credenciales JWT hardcodeadas en el portal Ciudadano | 🟩 | `docs/pieces/ciudadano/README.md §Autenticación` |
| 56 | Los patrones de disclosure de alcance, divulgación progresiva y hand-off | 🟦 | [../Antecedentes/IA-Mercado-Libre.md](../Antecedentes/IA-Mercado-Libre.md) |
| 57 | Convención de marcas y vocabulario del estudio | 🟩 | [../Antecedentes/Analisis-Asistencia-IA-ChatBotIA.md](../Antecedentes/Analisis-Asistencia-IA-ChatBotIA.md) |
| 58 | Toda la §3 (inventario), §5.2-§5.5 (procedimiento de sinónimos), §6 (tareas), §7.5 (prompt propuesto), §8 (banco), §9.3-§9.4 (ciclo), §10 (árbol), §12 (checklist) | 🟨 | **Propuesta de este estudio.** No hay práctica vigente que verificar: el asistente no está en producción (ítem 53). |

---

## Anexo A — Lo reusable para los próximos casos

> 🟨 Este es el **primer caso de éxito** y debe servir de modelo. Lo que sigue es lo que se copia tal cual a Multas, Habilitaciones, Reclamos o cualquier otra área.

| Activo | Reusable | Qué hay que cambiar |
|---|---|---|
| **El procedimiento de relevamiento de sinónimos (§5.2)** | ✅ **Al 100%** | Nada. Las 5 fuentes son las mismas en cualquier área. **Es el activo de mayor valor transferible.** |
| **El formato del documento de sinónimos (§5.3)** | ✅ Al 100% | El contenido |
| **La regla de acentos (§5.4)** | ✅ Al 100% | Verificar si el área nueva también tiene datos sin tildes (probable: misma base) |
| **Las 7 reglas de redacción (§4.1)** | ✅ Al 100% | Nada: derivan del motor, no del dominio |
| **El ciclo estándar de publicación (§6.0)** | ✅ Al 100% | Nada |
| **El árbol de diagnóstico (§10)** | ✅ ~90% | Las hojas de "sistema anfitrión" cambian según la app |
| **La estructura de la KB (§3.4)** | ✅ Al 100% | Los nombres de documento |
| **La separación en dos tenants por audiencia (§1.3)** | ✅ Si el área tiene 2 audiencias | Si sólo tiene público, un tenant |
| **El procedimiento de prompt (§7.4)** | ✅ Al 100% | El contenido del prompt |
| **La estructura del banco de regresión (§8)** | ✅ La estructura | Las preguntas son del dominio. 🟨 Pero el **patrón** se repite: siempre incluir casos de (a) sinónimo coloquial, (b) cosa que el sistema NO hace, (c) trámite inexistente, (d) pedido de acción, (e) dato personal, (f) info que la KB no tiene. |
| **Los checklists (§12)** | ✅ Al 100% | El disparador "reconciliar contra la tabla de catálogo" cambia de tabla |
| **La lista de "qué NO hacer" (§11)** | ✅ Al 100% | Nada |
| Inventario de documentos (§3.2/§3.3) | ❌ | Específico de Turnos |
| Contenido de los ejemplos (§4.2-§4.5) | ❌ | Específico de Turnos |

🟨 **La lección transferible de este caso, en una frase:** *el trabajo no es escribir lo que el sistema hace, sino escribir el puente entre cómo lo dice la gente y cómo lo llama el sistema* — y en un RAG léxico sin tabla de alias, ese puente sólo existe si el administrador lo escribe a mano.
