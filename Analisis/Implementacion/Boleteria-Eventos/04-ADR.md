# 04 · Architecture Decision Record (ADR) — Asistencia IA sobre Gestión de Eventos en Boletería Digital

> **Propósito.** Registrar, con contexto y evidencia citada, las decisiones de arquitectura que gobiernan el caso de éxito **"asistente de diagnóstico de configuración de eventos"** sobre `BoleteriaCore.Backoffice`, integrando el gateway **Ng-IAServices / IAConnect**.
>
> **Alcance.** Decisiones **específicas de este caso**. La metodología transversal del gateway (alta de tenant, edición de KB, function-calling genérico, operación) NO se repite acá: vive en [`../Ng-IAServices/01-SAD.md`](../Ng-IAServices/01-SAD.md), [`02-HLD.md`](../Ng-IAServices/02-HLD.md), [`03-LLD.md`](../Ng-IAServices/03-LLD.md), [`04-ADR.md`](../Ng-IAServices/04-ADR.md), [`05-Operations-Guide.md`](../Ng-IAServices/05-Operations-Guide.md), [`06-Administrator-Guide.md`](../Ng-IAServices/06-Administrator-Guide.md).
>
> **Audiencia.** Arquitectos y desarrolladores de NG-SA, responsable funcional de Boletería, operaciones, y quien deba aprobar o cancelar la continuidad del caso.
>
> **Estado del documento.** 🟨 **Propuesta.** Los 17 ADR de este archivo son **decisiones propuestas, ninguna implementada**. Donde se describe código existente, se cita con 🟩.
>
> **Convención de marcas** (heredada de [`../Antecedentes/Analisis-Asistencia-IA-ChatBotIA.md`](../Antecedentes/Analisis-Asistencia-IA-ChatBotIA.md)):
> · 🟩 **hecho verificado en fuente** (se cita ruta:línea) · 🟦 **práctica de industria establecida** · 🟨 **interpretación / inferencia propia**.
> Lo no confirmado se rotula **No verificado**.

---

## Tabla de contenidos

1. [Cómo leer este ADR](#1-cómo-leer-este-adr)
   1. [Formato de cada decisión](#11-formato-de-cada-decisión)
   2. [Numeración y alineación con el SAD](#12-numeración-y-alineación-con-el-sad)
   3. [Mapa de dependencias entre decisiones](#13-mapa-de-dependencias-entre-decisiones)
   4. [Las seis fuerzas transversales del caso](#14-las-seis-fuerzas-transversales-del-caso)
2. [ADR-001 — API adaptadora `BoleteriaCore.AI.Api` como capa de tools](#2-adr-001--api-adaptadora-boleteriacoreaiapi-como-capa-de-tools)
3. [ADR-002 — Deep-links devueltos por la tool, jamás construidos por el LLM](#3-adr-002--deep-links-devueltos-por-la-tool-jamás-construidos-por-el-llm)
4. [ADR-003 — Propagación de identidad: token-exchange de la cookie del Backoffice](#4-adr-003--propagación-de-identidad-token-exchange-de-la-cookie-del-backoffice)
5. [ADR-004 — Function-calling genérico en IAConnect, no un hack de boletería](#5-adr-004--function-calling-genérico-en-iaconnect-no-un-hack-de-boletería)
6. [ADR-005 — La regla de publicación se reimplementa en la tool, con test de equivalencia](#6-adr-005--la-regla-de-publicación-se-reimplementa-en-la-tool-con-test-de-equivalencia) ⭐
7. [ADR-006 — Arquitectura de conocimiento híbrida: RAG para lo estable, tools para lo volátil](#7-adr-006--arquitectura-de-conocimiento-híbrida-rag-para-lo-estable-tools-para-lo-volátil)
8. [ADR-007 — El asistente no ejecuta acciones: tools de sólo lectura en la v1](#8-adr-007--el-asistente-no-ejecuta-acciones-tools-de-sólo-lectura-en-la-v1)
9. [ADR-008 — Widget como componente Blazor en `MainLayout`, no script de CDN](#9-adr-008--widget-como-componente-blazor-en-mainlayout-no-script-de-cdn)
10. [ADR-009 — Dos tenants por perfil de usuario, no un system prompt condicional](#10-adr-009--dos-tenants-por-perfil-de-usuario-no-un-system-prompt-condicional)
11. [ADR-010 — ⚖️ El tenant de IAConnect mapea al perfil, no al municipio (resuelve incoherencia **C**)](#11-adr-010--️-el-tenant-de-iaconnect-mapea-al-perfil-no-al-municipio-resuelve-incoherencia-c)
12. [ADR-011 — Alcance del MVP: diagnosticar la cadena Evento→Función→FuncionUbicacion→Tarifa](#12-adr-011--alcance-del-mvp-diagnosticar-la-cadena-eventofunciónfuncionubicaciontarifa)
13. [ADR-012 — Stored procedures no verificables: se bloquea la capacidad, no se adivina](#13-adr-012--stored-procedures-no-verificables-se-bloquea-la-capacidad-no-se-adivina)
14. [ADR-013 — Curaduría y propiedad de la KB: dueño funcional + pipeline idempotente](#14-adr-013--curaduría-y-propiedad-de-la-kb-dueño-funcional--pipeline-idempotente)
15. [ADR-014 — Fallback ante proveedor LLM caído: degradación determinística, no failover](#15-adr-014--fallback-ante-proveedor-llm-caído-degradación-determinística-no-failover)
16. [ADR-015 — Medición del éxito y criterio de continuidad (go / no-go)](#16-adr-015--medición-del-éxito-y-criterio-de-continuidad-go--no-go)
17. [ADR-016 — ⚖️ Catálogo canónico de tools: T1…T6 (resuelve incoherencia **A**)](#17-adr-016--️-catálogo-canónico-de-tools-t1t6-resuelve-incoherencia-a)
18. [ADR-017 — ⚖️ Nomenclatura canónica del enum: `CausaNoPublicado` (resuelve incoherencia **B**)](#18-adr-017--️-nomenclatura-canónica-del-enum-causanopublicado-resuelve-incoherencia-b)
19. [Tabla resumen de ADRs](#19-tabla-resumen-de-adrs)
20. [Trazabilidad de evidencia](#20-trazabilidad-de-evidencia)

> ⚖️ **Los tres ADR marcados con balanza son decisiones de desempate.** Un análisis cruzado de los documentos ya escritos de este bloque detectó tres contradicciones reales entre `01-SAD.md`, `02-HLD.md`, `03-LLD.md` y `06-Administrator-Guide.md`. Este documento es el **decisor**: [`03-LLD.md`](03-LLD.md) y [`05-Operations-Guide.md`](05-Operations-Guide.md) se escriben después y **implementan lo que acá se resuelve**. Donde un documento previo contradiga a ADR-010, ADR-016 o ADR-017, **gana este documento** y el otro se corrige (🟩 es la tarea `T-0.5` del [`07-Plan-Sprints-Capacitacion.md`](07-Plan-Sprints-Capacitacion.md#270), que declara esta deuda como bloqueante del épico E-2).

---

## 1. Cómo leer este ADR

### 1.1 Formato de cada decisión

Estructura clásica de Nygard 🟦, extendida con un bloque de **Evidencia** obligatorio por la regla de trazabilidad del estudio:

| Sección | Qué contiene |
|---|---|
| **Contexto** | Las fuerzas del problema. Hechos 🟩 y marco 🟦; lo inferido va 🟨. |
| **Decisión** | Una oración imperativa. Sin condicionales. |
| **Alternativas consideradas** | Tabla: opción · atractivo · **por qué se descarta**. |
| **Consecuencias positivas** | Qué habilita. |
| **Consecuencias negativas** | Qué cuesta, qué deuda deja, qué se rompe. |
| **Estado** | Propuesto / Aceptado / Rechazado / Supersedido. Hoy: **todos Propuesto**. |
| **Evidencia** | Rutas concretas que sostienen el contexto. |

### 1.2 Numeración y alineación con el SAD

🟩 El [`01-SAD.md`](01-SAD.md) §3.4 ("matriz driver → decisión") ya referencia **ADR-01 … ADR-09** con un significado asignado. 🟩 **Ese mapeo no se conserva**: 8 de esas 9 referencias apuntan a decisiones distintas de las que este documento numera con el mismo ID. **Este documento fija la numeración canónica** (ADR-001…ADR-017) y **el `01-SAD.md` se corrige contra él**, no al revés (🟩 trabajo declarado en `T-0.5` del [`07-Plan-Sprints-Capacitacion.md`](07-Plan-Sprints-Capacitacion.md#270), bloqueante del épico E-2). La columna "Anclado por el SAD en" de la tabla siguiente indica **dónde el SAD toca el tema**, no que el ID coincida.

| ADR | Tema | Anclado por el SAD en |
|---|---|---|
| 001 | API adaptadora | §3.4 (DR-1, RA-4) · §5.3 |
| 002 | Deep-links por plantilla | §3.4 (DR-2, RA-2) · §6.4 |
| 003 | Autorización e identidad | §3.4 (DR-3, DR-4, RA-7, RA-8) · §9 |
| 004 | Function-calling genérico | §3.4 (RI-1, DR-8) |
| 005 | Regla reimplementada + test de equivalencia | §3.4 (DR-7, RA-6) |
| 006 | Arquitectura de conocimiento híbrida (RAG / tools) | §3.4 (RI-2, RI-3) · §8.3 |
| 007 | Tools de sólo lectura | §3.4 (§2.5, DR-3) |
| 008 | Widget en `MainLayout` | §3.4 (RA-1, RA-9, RA-10) · §6.1 |
| 009 | Dos tenants | §3.4 (RA-11) · §5.2 |
| 010 | ⚖️ Tenant ↔ perfil, no municipio | 🟨 nuevo acá — **supersede** `01-SAD.md` §6.6 |
| 011 | Alcance del MVP | 🟨 nuevo acá (formaliza §2.5) |
| 012 | Sprocs no verificables | 🟨 nuevo acá (formaliza RA-5) |
| 013 | Curaduría de KB | 🟨 nuevo acá |
| 014 | Fallback LLM caído | 🟨 nuevo acá |
| 015 | Go / no-go | 🟨 nuevo acá (formaliza CE-1…CE-8) |
| 016 | ⚖️ Catálogo canónico de tools | 🟨 nuevo acá — **supersede** `01-SAD.md` §6.3 |
| 017 | ⚖️ Nombre canónico del enum | 🟨 nuevo acá — **supersede** `02-HLD.md` §12.3 |

🟨 **Orden de lectura recomendado** ≠ orden numérico. Para entender el caso: **011** (qué se hace) → **005** (el corazón) → **001** (dónde vive) → **002** (el entregable) → **003** (quién puede) → el resto.

🟨 **Orden de lectura para implementar**: **016** y **017** primero. Son puro contrato: fijan los nombres con los que están escritos todos los demás ADR. Sin ellos, `03-LLD.md` no se puede escribir sin elegir por su cuenta — que es exactamente cómo se originó la divergencia.

### 1.3 Mapa de dependencias entre decisiones

```mermaid
flowchart TD
    A016["ADR-016 ⚖️<br/>Catálogo canónico<br/>de tools"] --> A004
    A017["ADR-017 ⚖️<br/>Enum canónico<br/>CausaNoPublicado"] --> A005
    A017 --> A002
    A011["ADR-011<br/>Alcance MVP"] --> A016
    A011 --> A009["ADR-009<br/>Dos tenants"]
    A009 --> A010["ADR-010 ⚖️<br/>Tenant ↔ perfil,<br/>no municipio"]
    A011 --> A004["ADR-004<br/>Function-calling<br/>genérico"]
    A004 --> A001["ADR-001<br/>API adaptadora"]
    A001 --> A005["ADR-005 ⭐<br/>Regla reimplementada<br/>+ test equivalencia"]
    A001 --> A003["ADR-003<br/>Token-exchange<br/>de identidad"]
    A005 --> A002["ADR-002<br/>Deep-links<br/>por la tool"]
    A003 --> A002
    A011 --> A007["ADR-007<br/>Sólo lectura"]
    A007 --> A002
    A011 --> A006["ADR-006<br/>Conocimiento híbrido<br/>RAG + tools"]
    A006 --> A013["ADR-013<br/>Curaduría de KB"]
    A005 -.->|"límite duro"| A012["ADR-012<br/>Sprocs opacos"]
    A011 --> A008["ADR-008<br/>Widget en MainLayout"]
    A004 --> A014["ADR-014<br/>Fallback LLM"]
    A011 --> A015["ADR-015<br/>Go / no-go"]
    A015 -.->|"puede revertir"| A011
    A012 -.->|"bloquea capacidades"| A011

    classDef fund fill:#e8f0fe,stroke:#3367d6,stroke-width:2px,color:#000
    classDef core fill:#fce8e6,stroke:#d93025,stroke-width:3px,color:#000
    classDef seg fill:#fef7e0,stroke:#f9ab00,stroke-width:2px,color:#000
    classDef ops fill:#e6f4ea,stroke:#137333,stroke-width:2px,color:#000
    classDef desemp fill:#f3e8fd,stroke:#8430ce,stroke-width:2px,color:#000
    class A011,A009,A004 fund
    class A005,A002 core
    class A001,A003,A007,A008 seg
    class A006,A012,A013,A014,A015 ops
    class A010,A016,A017 desemp
```

🟨 **Los tres nodos violeta no agregan capacidades: sacan ambigüedad.** Son baratos de decidir y carísimos de no decidir — 🟩 el plan de sprints los cuantifica en 8 puntos de `T-0.5` y los declara **precondición de todo el épico E-2** ([`07-Plan-Sprints-Capacitacion.md`](07-Plan-Sprints-Capacitacion.md#270)).

🟨 **Los dos nodos rojos son el caso.** ADR-005 decide **de dónde sale la verdad** del diagnóstico; ADR-002 decide **quién arma el enlace**. Si esos dos salen mal, el resto del documento es decoración: el asistente diría cosas falsas y llevaría a URLs inexistentes — exactamente los dos modos de falla que destruyen la confianza del usuario inexperto que el caso pretende ayudar.

### 1.4 Las seis fuerzas transversales del caso

Se enuncian una vez y se referencian después.

| # | Fuerza | Marca | Sustento |
|---|---|---|---|
| **F1** | **La cadena de datos tiene cuatro saltos y el precio vive en la tabla puente.** `Evento 1—N Función 1—N FuncionUbicacion N—N Tarifa`, con `Precio` en `sys_Tarifas_U_FuncionUbicacion`. `sys_Tarifas` **no tiene FK alguna**. | 🟩 | `SysTarifasModel.cs:11-33` · `SysTarifasUFuncionUbicacionModel.cs:17-19` · [`ia-db/indexes/02_Modelo-Dominio.md`](../../../../../BD/Boleteria.Core.Documentacion/ia-db/indexes/02_Modelo-Dominio.md) |
| **F2** | **"Publicado" no existe en la base.** Es propiedad de ViewModel que invierte `Pausado`, columna que **ni siquiera está mapeada** en el Model. No hay estado enum, ni borrador, ni `Fecha_Publicacion` de evento. | 🟩 | `ParametrosEventosEdit.razor.cs:174` (`Publicado = !Pausado`) · `SysVentaEntradasEventosModel.cs:57` (`Activo` mapeado; `Pausado` no) |
| **F3** | **La regla de publicación real es esencialmente UNA**: debe existir al menos una tarifa con `Precio > 0` en una función activa. El resto son validaciones de wizard o advertencias. | 🟩 | `ParametrosEventos.razor.cs:390-405` → modal `:422-436` · `ParametrosEventosEdit.razor.cs:1090-1105` |
| **F4** | **Toda la validación vive client-side, en code-behind Blazor.** No hay Service ni excepción de dominio que la cubra; las de `BoleteriaCore.Exceptions` son todas de compra/carrito/gateway. Y hay una **inconsistencia real**: en la misma pantalla `AccionCambiarEstado` valida tarifas y `AccionPausar` no. | 🟩 | grep exhaustivo sobre `Services/` y `Exceptions/` · `ParametrosEventos.razor.cs:386-420` vs. `:441-461` |
| **F5** | **IAConnect no tiene function-calling y su RAG es léxico TF-IDF.** Cero `tool_use`/`tool_choice`/`function_call`; `VectorEmbedding = null` siempre; `SerializeEmbedding` es código muerto. | 🟩 | grep exhaustivo sobre la solución IAConnect · `KnowledgeService.cs:75` · `RAGEngine.cs:34-127` |
| **F6** | **BoleteriaCore no tiene multi-tenant, ni ORM, ni tests, ni los cuerpos de sus sprocs.** Lo más cercano a tenant es `GP_IdMunicipio` y el parámetro `CONFIG_codMunicipio`. El repo tiene 2 archivos `.sql` (`issue-505`, `issue-506`). | 🟩 | `SysVentaEntradasEventosModel.cs:23` · `DataManager/Migraciones/` · [`ia-db ADR-0008`](../../../../../BD/Boleteria.Core.Documentacion/boleteria-core-docs/docs/04-decisions/) |

🟨 **Lectura de conjunto.** F1+F2+F3 dicen que **la respuesta que el usuario necesita es calculable** — es una consulta sobre cuatro tablas y un `> 0`. F4 dice que **esa consulta hoy no existe fuera de una pantalla**. F5 dice que **el gateway no puede consultar nada**. F6 dice que **no hay red de seguridad**. El caso, entonces, no es "prender un chatbot": es **construir la consulta que nadie escribió, en un lugar donde pueda vivir, y probar que dice lo mismo que la pantalla**. Todo lo que sigue es sobre dónde ponerla y cuánto riesgo aceptar.

---

## 2. ADR-001 — API adaptadora `BoleteriaCore.AI.Api` como capa de tools

### Contexto

🟩 IAConnect es un servicio **genérico y multi-tenant**: `lut_Tenants.Id_Tenant varchar(50)` es PK de negocio y raíz del particionado; el proveedor, el modelo, la temperatura y el system prompt son columnas del tenant (`scripts/01_create_database.sql:31-53`). 🟨 Su valor está justamente en no saber nada de ningún dominio. GDA-Turnos es su primer caso; boletería es el segundo (🟩 DR-8 del [`01-SAD.md`](01-SAD.md#31-drivers-arquitectónicos)).

🟩 BoleteriaCore, del otro lado, **no tiene ORM**: `DataEntityCore` compone el nombre del sproc por convención (`"sp_" + tabla + sufijo`) y **bindea posicionalmente** tras `SqlCommandBuilder.DeriveParameters` (`Notions.Core.Utils.DataManager/DataEntityCore.cs:18-27,43-46`). 🟨 Eso significa que **un cambio en el orden de parámetros de un sproc compila igual y rompe en runtime** (RA-4). Ese acoplamiento frágil es un motivo fuerte para no repartirlo entre repositorios.

🟩 Además, el estado que la tool necesita **no es leíble por el Model**: `Pausado` se lee crudo del `DataRow` (`Convert.ToBoolean(row["Pausado"])`, `ParametrosEventos.razor.cs:194,472`) porque `SysVentaEntradasEventosModel` no la mapea (RA-6, F2).

🟩 Y el driver duro es no reescribir el anfitrión: el wizard de alta tiene **6.212 líneas** en un solo code-behind (`ParametrosEventosAlta.razor.cs`), y la pieza completa suma 11.777 (`01-SAD.md` DR-1).

### Decisión

**Se crea un proyecto nuevo, `BoleteriaCore.AI.Api`, dentro de la solución de BoleteriaCore, que expone las tools por HTTP/JSON a IAConnect y reusa los DataManagers existentes. IAConnect NO referencia código de BoleteriaCore ni abre una conexión a su base.**

```mermaid
flowchart LR
    W["Widget en<br/>MainLayout 🟨"] -->|"chat"| IA["IAConnect 🟩<br/>+ tool loop 🟨"]
    IA -->|"POST /ai/tools/{nombre}<br/>JSON"| API["BoleteriaCore.AI.Api 🟨<br/>net10 · mismo repo"]
    API --> DM["DataManagers 🟩<br/>sin cambios"]
    DM --> SP["sprocs 🟩"]
    SP --> DB[("BD BoleteriaCore 🟩")]

    IA -.->|"❌ prohibido"| DB
    IA -.->|"❌ prohibido"| DM

    classDef prop fill:#fef7e0,stroke:#f9ab00,color:#000
    classDef ex fill:#e6f4ea,stroke:#137333,color:#000
    class W,API prop
    class IA,DM,SP,DB ex
```

### Alternativas consideradas

| Alternativa | Atractivo | Por qué se descarta |
|---|---|---|
| **A. IAConnect referencia `BoleteriaCore.DataManager`** | Cero red, cero contrato, cero latencia extra. La tool sería una llamada en proceso. | 🟩 **Rompe DR-8**: IAConnect dejaría de ser genérico — cada dominio nuevo agregaría una referencia de proyecto al gateway compartido. 🟩 **Y hay un choque de runtime**: los hosts de BoleteriaCore son net10 (`01-SAD.md` RA-1) e IAConnect vive en su propia solución. 🟨 Peor: IAConnect **no conoce los perfiles del Backoffice** (`parametros`, `hacienda`, `control-acceso`, leídos de `sys_BackOffice_UsuariosPerfiles`, [`ia-db/08_Seguridad.md`](../../../../../BD/Boleteria.Core.Documentacion/ia-db/indexes/08_Seguridad.md)), así que la autorización terminaría reimplementada del lado equivocado. |
| **B. API adaptadora en BoleteriaCore** ✅ | Contrato JSON explícito; cada uno en su runtime; la autorización se valida contra la identidad real del BO; RA-4 queda contenido en el mismo repo, misma pipeline. | — **Elegida.** |
| **C. El widget resuelve el diagnóstico y se lo pasa al LLM en el prompt** | Bajísimo acoplamiento; la identidad ya está resuelta del lado cliente; cero API nueva. | 🟨 **Rompe DR-2 y deja de ser un asistente**: el LLM no *decide* qué consultar, recibe un blob fijo. Un flujo fijo con redacción bonita no escala a "¿y a qué función le falta el precio?" ni a "¿cuántas entradas vendí?". 🟩 Además el contenido inyectado iría a `PromptBuilder`, que **no escapa nada** (`PromptBuilder.cs:10-55`, RI-10): meter datos de negocio crudos en el prompt es superficie de injection. |
| **D. Tools ejecutadas como sprocs directos desde IAConnect** | Sin proyecto nuevo; IAConnect ya sabe hablar con SQL Server vía `DataEntityCore`. | 🟩 **Segunda cadena de conexión al dominio** — `DataEntityCore.Configure()` es un **singleton estático con una sola connection string** (`IAConnect.API/Program.cs:22` + `DataEntityCore.cs:33-256`): IAConnect **no puede** tener dos bases sin refactor. 🟩 Y sin identidad del BO, la autorización sería imposible (DR-4). ❌ |

### Consecuencias positivas

- 🟨 **El diff sobre el código existente es de una línea** (el `<AsistenteWidget />` en `MainLayout.razor`). DR-1 cumplido de forma verificable, no declamada.
- 🟨 **La autorización se impone donde están los datos para imponerla.** La API vive en el repo que conoce `sys_BackOffice_UsuariosPerfiles` y `IdBotonPago`.
- 🟩 **RA-4 queda contenido.** Si cambia el orden de parámetros de un sproc, rompe en el mismo repo, misma pipeline, mismos tests que el sproc — no en un servicio ajeno que nadie va a correlacionar.
- 🟨 **El molde es reusable.** El próximo dominio pone su propia API adaptadora con su propio corte de autorización; IAConnect no cambia.

### Consecuencias negativas

- 🟨 **Un proyecto más que desplegar, monitorear y versionar.** Y un salto de red más en el camino crítico del diagnóstico (DR-5, latencia).
- 🟨 **Un contrato más que puede divergir.** El JSON Schema de las tools (ADR-004) y la implementación pueden separarse. Mitigación: `ToolSchemaProvider` genera el schema desde los tipos, no a mano.
- 🟩 **Mockear es difícil.** La capa de servicios de BoleteriaCore **no tiene interfaces**: 33 archivos, inyección por tipo concreto, sin `ICarritoService` (RA-12). Los tests de la API adaptadora (ADR-005) van a tener que golpear base o envolver los DataManagers.
- 🟨 **Deuda deliberada:** la API adaptadora es un segundo lugar donde vive lógica de negocio de eventos. ADR-005 se hace cargo de eso explícitamente.

### Estado

🟨 **Propuesto.** Depende de [ADR-004](#5-adr-004--function-calling-genérico-en-iaconnect-no-un-hack-de-boletería). Habilita [ADR-003](#4-adr-003--propagación-de-identidad-token-exchange-de-la-cookie-del-backoffice) y [ADR-005](#6-adr-005--la-regla-de-publicación-se-reimplementa-en-la-tool-con-test-de-equivalencia).

### Evidencia

| Afirmación | Fuente |
|---|---|
| `DataEntityCore` compone el sproc por convención y bindea posicionalmente | `Notions/Notions.Core.Utils.DataManager/DataEntityCore.cs:18-27,43-46` |
| `DataEntityCore.Configure()` de IAConnect es singleton estático con una sola connection string | `IAConnect.API/Program.cs:22` + `IAConnect.Infrastructure/DataAccess/DataEntityCore.cs:33-256` |
| `Pausado` se lee cruda del `DataRow`, no está en el Model | `ParametrosEventos.razor.cs:194,472` · `SysVentaEntradasEventosModel.cs:57` |
| `PromptBuilder` interpola contenido sin escapado | `IAConnect.Application/Services/PromptBuilder.cs:10-55` |
| Perfiles del BO en `sys_BackOffice_UsuariosPerfiles`, comprobados con `TienePerfil()` | [`ia-db/indexes/08_Seguridad.md`](../../../../../BD/Boleteria.Core.Documentacion/ia-db/indexes/08_Seguridad.md) · `MainLayout.razor.cs:79` |
| Capa de servicios sin interfaces (RA-12) | [`01-SAD.md §3.2`](01-SAD.md#32-restricciones-duras-del-anfitrión-boleteriacore) |
| `lut_Tenants` es raíz del particionado de IAConnect | `IAConnect/scripts/01_create_database.sql:31-53` |

---

## 3. ADR-002 — Deep-links devueltos por la tool, jamás construidos por el LLM

### Contexto

🟩 **El deep-link es el entregable del caso.** La definición textual del usuario lo dice: *«Incluso generar un enlace puntual a la página donde configurar ese parámetro que faltó.»* Sin enlace, el asistente repite el texto del modal que el usuario ya vio y no entendió (`01-SAD.md` §2.3).

🟩 **Y el terreno lo habilita**: todas las rutas del Backoffice son **planas con query string** — ninguna página usa parámetros de ruta, todo entra por `[SupplyParameterFromQuery]` (RA-2, `ParametrosEventosEdit.razor:1`). El enlace es literalmente `/{Pagina}?{param}={id}`.

🟨 **Pero eso mismo es la trampa.** Una plantilla tan simple es *exactamente* lo que un LLM alucina bien: sintácticamente perfecta, semánticamente inventada. Y hay tres pruebas concretas de que el espacio de rutas de este Backoffice **castiga** al que adivina:

| Trampa | Evidencia | Qué pasa si el LLM adivina |
|---|---|---|
| **`/ParametrosEventosEditFunciones` tiene dos firmas incompatibles** | 🟩 Tres `[SupplyParameterFromQuery]` (`ParametrosEventosEditFunciones.razor.cs:24,26,28`), invocada como `?idFuncion={id}` **y** como `?idEvento={id}&idLugar={id}` (RA-3) | Abre la pantalla en modo "crear función" cuando el usuario necesitaba "editar precios". El link carga (HTTP 200) y **está mal**. Es el peor caso: falla silenciosa. |
| **`ParametrosMapasCoordenadas` no tiene `@page`** | 🟩 Tiene `@rendermode` y `[Authorize]` pero **no `@page`** (`ParametrosMapasCoordenadas.razor:1-3`); el wizard igual navega a `ParametrosMapasCoordenadas?IdL=...` (`ParametrosEventosAlta.razor.cs:3483-3487`) | 404. Y el LLM tiene toda la razón "lógica" para inventarla: existe el componente, existe el concepto, existe el error de validación que la necesita. |
| **`/hacienda-evento` no existe** | 🟩 `AuthController.cs#L72` redirige ahí con `tipo=eventual`, y **no hay ningún `@page "/hacienda-evento"`** entre las 38 rutas del host ([`routes-map.md`](../../../../../BD/Boleteria.Core.Documentacion/boleteria-core-docs/docs/pieces/boleteria-core-backoffice/routes-map.md)) | El propio sistema ya tiene un link roto en producción. Un LLM que "razone por analogía" sobre el código va a reproducir el bug. |

🟦 **Y hay un vector de seguridad, no sólo de calidad.** OWASP LLM01 (prompt injection) + LLM02 (insecure output handling): 🟩 `PromptBuilder` interpola los fragmentos de la KB **sin escapado alguno** (`PromptBuilder.cs:10-55`). Un documento de KB envenenado puede inducir al modelo a emitir una URL arbitraria — externa, de phishing — que el widget renderizaría como un enlace institucional con el estilo del Backoffice.

### Decisión

**El deep-link se construye exclusivamente en `DeepLinkBuilder`, dentro de `BoleteriaCore.AI.Api`, a partir de plantillas `const` y del `CausaNoPublicado` que devuelve el diagnóstico (⚖️ corregido por ADR-017). La tool retorna el link ya armado como campo estructurado (`deepLink: {url, texto}`). El LLM sólo puede *transcribir* ese campo; tiene prohibido componer URLs. El widget renderiza enlaces únicamente contra una allowlist de rutas conocidas, y descarta cualquier URL que no venga del campo `deepLink`.**

🟨 Snippet **propuesto** (no existe en el repo) — las plantillas verificadas están en [`01-SAD.md §6.4`](01-SAD.md#64-deeplinkbuilder--plantillas-verificadas):

```csharp
// 🟨 PROPUESTA — BoleteriaCore.AI.Api/Services/DeepLinkBuilder.cs
// Rutas verificadas contra ParametrosEventosEdit.razor.cs:260,1055-1083.
// Un test de CI las contrasta con las declaraciones @page reales (CE-2).
public static class DeepLinkBuilder
{
    private const string EditarFuncion = "/ParametrosEventosEditFunciones?idFuncion={0}";  // 🟩 :1065
    private const string CrearFuncion  = "/ParametrosEventosEditFunciones?idEvento={0}&idLugar={1}"; // 🟩 :260
    private const string HubEvento     = "/ParametrosEventosEdit?idEvento={0}";            // 🟩 razor:1
    private const string Lugares       = "/ParametrosEventosEditLugares?idEvento={0}";     // 🟩 :1069

    public static DeepLink? Build(CausaNoPublicado causa, DiagnosticoContexto ctx) => causa switch
    {
        CausaNoPublicado.TarifasSinPrecio  => new(string.Format(EditarFuncion, ctx.IdPrimeraFuncionActiva),
                                                  "Cargar precios en las tarifas"),
        CausaNoPublicado.SinFunciones      => new(string.Format(CrearFuncion, ctx.IdEvento, ctx.IdLugar),
                                                  "Crear la primera función"),
        CausaNoPublicado.FuncionesInactivas=> new(string.Format(HubEvento, ctx.IdEvento), "Activar una función"),
        CausaNoPublicado.SinUbicaciones    => new(string.Format(Lugares, ctx.IdEvento), "Asignar ubicaciones"),

        // ⚠ 🟩 ParametrosMapasCoordenadas NO tiene @page: no hay destino navegable.
        // Se devuelve null a propósito: el asistente describe la ruta manual y NO emite link.
        CausaNoPublicado.MapaSinCoordenadas => null,
        _ => null
    };
}
```

### Alternativas consideradas

| Alternativa | Atractivo | Por qué se descarta |
|---|---|---|
| **El LLM construye la URL** desde plantillas que están en la KB | Cero código. La KB ya va a tener el mapa de pantallas (`05-mapa-de-pantallas.md`, `01-SAD.md` §6.5). El modelo es bueno interpolando. | 🟨 **Es alucinación garantizada, no probable.** Tres razones 🟩: (1) RA-3 — dos firmas incompatibles para la misma ruta: el modelo no tiene forma de saber cuál corresponde a "cargar precios" salvo que se lo digamos, y si se lo decimos, ya decidimos nosotros; (2) `ParametrosMapasCoordenadas` es una ruta que *debería* existir y no existe: el LLM va a inventarla porque es lo razonable; (3) 🟩 el RAG es TF-IDF con top-5 y **sin threshold** (`RAGEngine.cs:34-120`) — el fragmento con la plantilla correcta puede simplemente no recuperarse, y el modelo va a completar igual. 🟦 Regla de industria: **la URL es una decisión, no una redacción**. |
| **El LLM construye la URL y la API la valida** (guardrail de salida) | Conserva la flexibilidad y agrega una red. | 🟨 Valida la *forma*, no la *intención*. `/ParametrosEventosEditFunciones?idEvento=42&idLugar=7` pasa cualquier validación de allowlist y de HTTP 200, y **abre la pantalla equivocada** cuando el problema era un precio faltante. Un validador que no puede detectar el modo de falla más común no es una red. Y agrega complejidad para conservar una libertad que no queremos. |
| **Deep-links en la KB como texto, sin tool** | Funciona para las rutas sin parámetros (`/ParametrosEventos`, `/Parametros`). | 🟨 Resuelve el 10% fácil. El caso de éxito **necesita el `{idFuncion}`**, y ese ID sale de recorrer la cadena F1 en la base — no hay KB estática que lo contenga (`01-SAD.md` §2.3, fila "Dame el link para arreglarlo"). |
| **Devolver sólo el `CausaNoPublicado` y que el widget arme el link** | El widget es Blazor y tiene `NavigationManager`: sabría armar rutas relativas correctas. | 🟨 Atractivo, y es un fallback válido. Se descarta como primaria porque el widget no tiene el `idFuncion` (lo tiene la tool) y porque duplicaría el `switch` en dos lugares — reproduciendo el problema de ADR-005 en chiquito. **Sí se adopta parcialmente**: el widget mantiene la **allowlist** de rutas como segunda barrera. |

### Consecuencias positivas

- 🟨 **CE-2 se vuelve testeable en CI.** Un test compara las plantillas `const` contra las 38 declaraciones `@page` reales (🟩 [`routes-map.md`](../../../../../BD/Boleteria.Core.Documentacion/boleteria-core-docs/docs/pieces/boleteria-core-backoffice/routes-map.md)). Si alguien renombra una ruta, la build rompe **antes** que el asistente mienta.
- 🟩 **El caso "sin destino" se maneja con honestidad.** `MapaSinCoordenadas => null` es una decisión explícita: 🟨 **emitir un link roto es peor que no emitir ninguno**, porque destruye la confianza que CE-2 mide. El asistente describe la ruta manual por UI.
- 🟦 **Cierra LLM02.** El widget nunca renderiza una URL que el modelo escribió. El vector de phishing vía KB envenenada queda cortado en la capa de render, no en el prompt.
- 🟨 **El veredicto es determinista.** `CausaNoPublicado` + `deepLink` no dependen de la temperatura del tenant.

### Consecuencias negativas

- 🟨 **Rigidez.** Una pantalla nueva exige código nuevo en `DeepLinkBuilder` y un deploy de la API. No alcanza con editar la KB. 🟦 Es el precio conocido de sacar decisiones del prompt.
- 🟨 **El `switch` de causas es una tercera copia del conocimiento del dominio** (junto con la UI y la KB). Se acota a un `enum` chico y se cubre con el test de equivalencia de ADR-005.
- 🟩 **RA-3 sigue siendo una decisión humana.** Elegir `?idFuncion=` vs. `?idEvento=&idLugar=` es una regla que escribimos nosotros, y si la escribimos mal, el link está mal con la misma prolijidad que si lo hubiera inventado el LLM. La diferencia es que **es un bug reproducible y testeable**, no una alucinación estocástica.
- 🟨 **PathBase.** 🟩 Las rutas del BO se sirven bajo un `PathBase` obligatorio ([`routes-map.md`](../../../../../BD/Boleteria.Core.Documentacion/boleteria-core-docs/docs/pieces/boleteria-core-backoffice/routes-map.md), "Cómo leer las tablas"). Las plantillas son **relativas**; el prefijo lo resuelve el widget con `NavigationManager`, nunca la API. **No verificado**: el valor del PathBase en cada ambiente.

### Estado

🟨 **Propuesto.** Depende de [ADR-001](#2-adr-001--api-adaptadora-boleteriacoreaiapi-como-capa-de-tools) y [ADR-005](#6-adr-005--la-regla-de-publicación-se-reimplementa-en-la-tool-con-test-de-equivalencia). Alineado con [`../Ng-IAServices/04-ADR.md`](../Ng-IAServices/04-ADR.md) ADR-018 (deep-links como contrato).

### Evidencia

| Afirmación | Fuente |
|---|---|
| Rutas planas con query string; sin parámetros de ruta (RA-2) | `ParametrosEventosEdit.razor:1` · [`routes-map.md`](../../../../../BD/Boleteria.Core.Documentacion/boleteria-core-docs/docs/pieces/boleteria-core-backoffice/routes-map.md) |
| `/ParametrosEventosEditFunciones` con tres query params y dos firmas de invocación (RA-3) | `ParametrosEventosEditFunciones.razor.cs:24,26,28` · `ParametrosEventosEdit.razor.cs:260,1065` |
| `ParametrosMapasCoordenadas` sin `@page`, pero navegada por el wizard | `ParametrosMapasCoordenadas.razor:1-3` · `ParametrosEventosAlta.razor.cs:3483-3487` |
| `/hacienda-evento` es destino de redirect y no existe entre las 38 rutas | `Backoffice/Controllers/AuthController.cs#L72` · [`routes-map.md`](../../../../../BD/Boleteria.Core.Documentacion/boleteria-core-docs/docs/pieces/boleteria-core-backoffice/routes-map.md) |
| Plantillas de deep-link verificadas | `ParametrosEventosEdit.razor.cs:260,1055-1083` |
| RAG top-5 sin threshold | `IAConnect.Application/Services/RAGEngine.cs:34-120` |
| `PromptBuilder` sin escapado (RI-10) | `IAConnect.Application/Services/PromptBuilder.cs:10-55` |
| Patrón de deep-link y disclosure de alcance | [`../Antecedentes/IA-Mercado-Libre.md`](../Antecedentes/IA-Mercado-Libre.md) |

---

## 4. ADR-003 — Propagación de identidad: token-exchange de la cookie del Backoffice

### Contexto

🟩 **El anfitrión autoriza por perfil, no por fila.** El Backoffice comprueba pertenencia a perfiles (`parametros`, `hacienda`, `control-acceso`) con `TienePerfil()` sobre `sys_BackOffice_UsuariosPerfiles` (`MainLayout.razor.cs:79`, [`ia-db/08_Seguridad.md`](../../../../../BD/Boleteria.Core.Documentacion/ia-db/indexes/08_Seguridad.md)). 🟨 Es autorización **de menú**: dice si ves el módulo Eventos, no *qué eventos* ves.

🟩 **Y no hay tenant.** No existe discriminador de aislamiento; lo más cercano es `GP_IdMunicipio` (`SysVentaEntradasEventosModel.cs:23`) y el parámetro global `CONFIG_codMunicipio`. 🟨 La segmentación *parece* ser por municipio, pero **no hay código que lo confirme como frontera de seguridad** (🟩 [`../Antecedentes/Relevamiento-Verificacion-BoleteriaCore.md:167-169`](../Antecedentes/Relevamiento-Verificacion-BoleteriaCore.md#no-verificado--límites-de-esta-verdad-de-referencia), bullet "Multi-tenant"). Ver [ADR-010](#11-adr-010--️-el-tenant-de-iaconnect-mapea-al-perfil-no-al-municipio-resuelve-incoherencia-c).

🟨 El riesgo concreto: si la tool acepta `idEvento` y lo consulta sin más, **cualquier usuario autenticado puede diagnosticar cualquier evento de cualquier municipio** enumerando enteros. Es un IDOR clásico (🟦 OWASP API1: Broken Object Level Authorization) servido por un chatbot — y el LLM, que es controlable por el prompt del propio usuario, **no puede ser el que decida** si el acceso corresponde.

### Decisión

**La identidad viaja como un JWT de vida corta emitido por el Backoffice mediante token-exchange, jamás como parámetro de la tool. `BoleteriaCore.AI.Api` deriva usuario y alcance *exclusivamente* de ese token, y valida todo `idEvento` contra el alcance antes de tocar el dominio. IAConnect transporta el token de forma opaca: no lo lee, no lo emite, no lo interpreta.**

```mermaid
sequenceDiagram
    autonumber
    participant U as Usuario BO
    participant W as Widget 🟨
    participant BO as Backoffice 🟩<br/>(cookie auth)
    participant IA as IAConnect 🟩
    participant API as AI.Api 🟨
    participant DB as BD 🟩

    U->>W: "¿por qué no se publicó?"
    W->>BO: token-exchange (cookie del circuito)
    BO-->>W: JWT corto {sub, perfiles, GP_IdMunicipio}<br/>aud=BoleteriaCore.AI.Api · exp=5min
    W->>IA: POST /chat {mensaje, authToken}
    Note over IA: 🟨 transporte opaco:<br/>no valida ni lee el JWT
    IA->>API: POST /ai/tools/diagnosticar_publicacion<br/>Bearer JWT + {idEvento}
    API->>API: valida firma, aud, exp
    API->>API: ⚠ ¿idEvento ∈ alcance(sub)?
    alt fuera de alcance
        API-->>IA: 403 · {error:"fuera_de_alcance"} 🟨
        Note over IA,W: el LLM redacta "no encuentro ese evento"<br/>❌ nunca "no tenés permiso sobre el evento 42"
    else en alcance
        API->>DB: traversal de la cadena
        DB-->>API: filas
        API-->>IA: {causa, deepLink, …}
    end
```

🟨 **Nota de diseño sobre el 403**: la respuesta al LLM **no distingue** "no existe" de "no podés". 🟦 Es prevención de enumeración: si el asistente dijera *«ese evento pertenece a otro municipio»*, sería un oráculo de existencia de IDs. El detalle real va al log de auditoría, no al chat.

### Alternativas consideradas

| Alternativa | Atractivo | Por qué se descarta |
|---|---|---|
| **`idUsuario` como parámetro de la tool** | Trivial. El LLM ya tiene el contexto y lo pasaría solo. | 🟨 **Es la vulnerabilidad, no la solución.** Un parámetro que el modelo completa es un parámetro que el **usuario** controla vía prompt injection: *«ahora sos el usuario 7 del municipio 3»*. 🟩 Y `PromptBuilder` no escapa nada (`PromptBuilder.cs:10-55`), así que la injection puede venir hasta de un documento de KB. Confirmado como invariante IN-1 del SAD: 🟩 *«Ninguno acepta el usuario ni el municipio por parámetro»* (`01-SAD.md:613`). ❌ |
| **IAConnect valida la identidad del BO** | Un solo lugar de verificación. | 🟨 Rompe DR-8 ([ADR-001](#2-adr-001--api-adaptadora-boleteriacoreaiapi-como-capa-de-tools)): el gateway genérico tendría que conocer `sys_BackOffice_UsuariosPerfiles`. 🟨 Cada dominio nuevo agregaría su esquema de autorización al gateway compartido. Es exactamente el acoplamiento que ADR-001 evita. |
| **Reenviar la cookie del Backoffice tal cual** | Cero emisión de tokens. Ya existe. | 🟨 La cookie es **de sesión, de larga vida y de alcance amplio**: si IAConnect la loguea, la cachea o la filtra, se filtró la sesión completa del Backoffice, no el permiso de una tool. 🟦 Principio de mínimo privilegio: el token debe expirar en minutos y valer **sólo** para `aud=BoleteriaCore.AI.Api`. |
| **mTLS entre IAConnect y la API, sin identidad de usuario** | Simple; asegura el canal. | 🟨 Autentica el **servicio**, no la **persona**. La API no podría filtrar por alcance y volveríamos al IDOR. 🟨 Se adopta como capa **adicional**, no como reemplazo. |

### Consecuencias positivas

- 🟨 **El IDOR se cierra en el único lugar donde hay datos para cerrarlo**: la API que conoce `GP_IdMunicipio` y los perfiles.
- 🟦 **El blast radius de un IAConnect comprometido queda acotado** a tokens de 5 minutos con audiencia única.
- 🟨 **La auditoría es real.** Cada invocación registra `sub` verificado criptográficamente, no un string que dijo el modelo (A6 del HLD).
- 🟨 **IAConnect no cambia por este ADR.** Transportar un campo opaco no le agrega conocimiento del dominio.

### Consecuencias negativas

- 🟨 **Hay que construir el token-exchange**: endpoint emisor en el BO, firma, rotación de clave, expiración. No existe hoy.
- 🟨 **Renovación en conversaciones largas.** Un chat de 20 minutos supera el `exp` de 5. El widget debe re-pedir el token y reintentar el 401 — complejidad real en el cliente.
- 🟨 **`alcance(sub)` no está definido en el origen.** 🟩 Como BoleteriaCore no tiene aislamiento por fila, hay que **inventar** la regla de alcance (probablemente `GP_IdMunicipio` del usuario). 🟨 Es una decisión de seguridad tomada sobre un sistema que nunca la tomó, y **puede no coincidir con lo que el Backoffice deja ver hoy por pantalla**. Riesgo declarado: el asistente podría ser *más restrictivo* que la UI. Se prefiere ese error al inverso.
- 🟩 **No verificado**: si `GP_IdMunicipio` es efectivamente el criterio de segmentación. La verdad de referencia lo marca como 🟨 inferencia ([`../Antecedentes/Relevamiento-Verificacion-BoleteriaCore.md:167-169`](../Antecedentes/Relevamiento-Verificacion-BoleteriaCore.md#no-verificado--límites-de-esta-verdad-de-referencia)). **Es la primera pregunta al responsable funcional**, y bloquea el diseño de `alcance()`.

### Estado

🟨 **Propuesto.** Depende de [ADR-001](#2-adr-001--api-adaptadora-boleteriacoreaiapi-como-capa-de-tools). Habilita [ADR-002](#3-adr-002--deep-links-devueltos-por-la-tool-jamás-construidos-por-el-llm) y [ADR-007](#8-adr-007--el-asistente-no-ejecuta-acciones-tools-de-sólo-lectura-en-la-v1). Relacionado con [ADR-010](#11-adr-010--️-el-tenant-de-iaconnect-mapea-al-perfil-no-al-municipio-resuelve-incoherencia-c).

### Evidencia

| Afirmación | Fuente |
|---|---|
| El BO autoriza por perfil con `TienePerfil()` sobre `sys_BackOffice_UsuariosPerfiles` | `MainLayout.razor.cs:79` · [`ia-db/indexes/08_Seguridad.md`](../../../../../BD/Boleteria.Core.Documentacion/ia-db/indexes/08_Seguridad.md) |
| No hay discriminador de tenant; lo más cercano es `GP_IdMunicipio` y `CONFIG_codMunicipio` | `SysVentaEntradasEventosModel.cs:23` · [`../Antecedentes/Relevamiento-Verificacion-BoleteriaCore.md:167-169`](../Antecedentes/Relevamiento-Verificacion-BoleteriaCore.md#no-verificado--límites-de-esta-verdad-de-referencia) |
| Invariante IN-1: ninguna tool acepta usuario ni municipio por parámetro | [`01-SAD.md:613`](01-SAD.md) §6.2, §10.3 |
| `PromptBuilder` interpola sin escapado ⇒ injection posible desde la KB | `IAConnect.Application/Services/PromptBuilder.cs:10-55` |
| Auditoría de invocaciones (regla A6) | [`02-HLD.md:1638`](02-HLD.md) §12.4 |

---

## 5. ADR-004 — Function-calling genérico en IAConnect, no un hack de boletería

### Contexto

🟩 **IAConnect hoy no puede llamar tools.** Grep exhaustivo sobre la solución: cero ocurrencias de `tool_use`, `tool_choice`, `function_call`. 🟩 Su RAG es léxico: `VectorEmbedding` es `null` siempre (`KnowledgeService.cs:75`) y `SerializeEmbedding` es código muerto; `RAGEngine` hace TF-IDF top-5 **sin threshold** (`RAGEngine.cs:34-127`). Fuerza **F5**.

🟨 Esto es *la* brecha del caso. Todo lo demás de este documento asume que el gateway puede invocar `diagnosticar_publicacion`. Hoy **no puede**, y ninguna KB por buena que sea lo suple: 🟩 la respuesta depende de filas concretas (`Precio > 0` en una función activa de un evento puntual), no de texto.

🟩 GDA-Turnos enfrentó la misma brecha y la resolvió en el gateway ([`../GDA-Turnos/04-ADR.md`](../GDA-Turnos/04-ADR.md) ADR-006: *«construir la capa de tools en IAConnect»*). 🟨 Boletería es el **segundo** consumidor: si cada caso construye su propio mecanismo, hay dos loops de tool-calling divergentes en el mismo servicio.

### Decisión

**El tool-calling se implementa como capacidad genérica de IAConnect, declarativa y dirigida por configuración de tenant: un `ToolRegistry` por tenant, con endpoint y JSON Schema, y un tool-loop acotado en el orquestador. IAConnect no contiene una sola línea de código que mencione eventos, funciones, tarifas ni boletería.**

🟨 El criterio de aceptación es negativo y verificable: **`grep -ri "evento\|tarifa\|boleteria" IAConnect/src/` debe dar 0 hits**. Si da uno, la capa no es genérica y este ADR está violado.

```mermaid
flowchart TD
    subgraph IA["IAConnect 🟨 extendido — CERO conocimiento de dominio"]
        O["ChatOrchestrator"] --> TR["ToolRegistry<br/>por tenant"]
        TR --> TC["ToolCaller<br/>HTTP + JSON Schema"]
        O --> LOOP{"¿tool_calls<br/>en la respuesta?"}
        LOOP -->|"sí · máx 3 vueltas 🟨"| TC
        LOOP -->|"no"| R["respuesta final"]
    end
    TC -->|"HTTP"| B["BoleteriaCore.AI.Api 🟨"]
    TC -->|"HTTP"| G["GDA-Turnos.AI.Api 🟨"]
    TC -.->|"futuro: sin tocar IAConnect"| N["Dominio N"]

    classDef gen fill:#e8f0fe,stroke:#3367d6,color:#000
    classDef dom fill:#fef7e0,stroke:#f9ab00,color:#000
    class O,TR,TC,LOOP,R gen
    class B,G,N dom
```

| Aspecto del loop | Decisión | Marca |
|---|---|---|
| Máximo de vueltas | **3**. Superado ⇒ se corta y se responde con lo obtenido | 🟨 |
| Timeout por tool | **3 s**. Vencido ⇒ degradación de [ADR-014](#15-adr-014--fallback-ante-proveedor-llm-caído-degradación-determinística-no-failover) | 🟨 |
| Tools por tenant | Filas de configuración, no `switch` en código | 🟨 |
| Registro del catálogo | Nombres exactos de [ADR-016](#17-adr-016--️-catálogo-canónico-de-tools-t1t6-resuelve-incoherencia-a) | ⚖️ |
| Errores de la tool | Se devuelven **tipados** al modelo; nunca se propaga un stack trace | 🟨 |

### Alternativas consideradas

| Alternativa | Atractivo | Por qué se descarta |
|---|---|---|
| **Hardcodear las tools de boletería en IAConnect** | El camino más corto a una demo. Una semana menos. | 🟩 **Rompe DR-8 explícitamente.** 🟨 Y es irreversible en la práctica: cuando el tercer dominio llegue, habrá dos implementaciones y nadie va a refactorizar la primera. La deuda se paga una vez acá o tres veces después. ❌ |
| **Sólo RAG, sin tools** | IAConnect ya lo tiene. Cero desarrollo en el gateway. | 🟩 **No puede responder la pregunta del caso.** *«¿Por qué no se publicó ESTE evento?»* exige leer filas. 🟩 Peor: TF-IDF top-5 sin threshold **siempre devuelve algo** (`RAGEngine.cs:34-120`), así que el modelo recibiría fragmentos irrelevantes y **redactaría una causa plausible y falsa**. Es la peor falla posible para CE-8. ❌ Ver [ADR-006](#7-adr-006--arquitectura-de-conocimiento-híbrida-rag-para-lo-estable-tools-para-lo-volátil). |
| **MCP (Model Context Protocol) como capa de tools** | 🟦 Estándar abierto; herramientas y ecosistema; el molde ya está diseñado. | 🟨 Se descarta **para la v1, no por mérito**: IAConnect es multi-proveedor y la normalización MCP↔proveedor agrega una capa que el caso no necesita todavía. 🟨 **Candidato explícito de v2**; el `ToolRegistry` se diseña con esa migración en mente (endpoint + schema es, a propósito, la forma de un server MCP). |
| **Tool-loop en el widget (cliente orquesta)** | Sin cambios en IAConnect. La identidad ya está del lado cliente. | 🟨 Pone la decisión de qué invocar en un cliente que el usuario controla. 🟦 Y obliga a exponer el catálogo de tools al navegador. ❌ |

### Consecuencias positivas

- 🟨 **El costo se amortiza entre dominios.** GDA-Turnos y boletería comparten el mecanismo; el tercero es gratis.
- 🟨 **Boletería no espera a que IAConnect "sepa" de boletería.** Registrar un tenant nuevo con sus tools es configuración.
- 🟦 **El límite de 3 vueltas acota el costo y el peor caso de latencia**, que en un loop sin tope es no acotado.
- 🟨 **Habilita el molde de [ADR-001](#2-adr-001--api-adaptadora-boleteriacoreaiapi-como-capa-de-tools)**: cada dominio expone HTTP+JSON y nadie toca el gateway.

### Consecuencias negativas

- 🟨 **Es el ítem más caro del plan y está en el camino crítico.** Nada del caso funciona sin esto; y lo construye un equipo que no es el dueño de boletería.
- 🟨 **Riesgo de coordinación entre bloques.** Si GDA-Turnos y boletería especifican el `ToolRegistry` en paralelo, divergen. Mitigación: el contrato lo fija [`../Ng-IAServices/03-LLD.md`](../Ng-IAServices/03-LLD.md), y este bloque lo **consume**, no lo define.
- 🟩 **No todos los proveedores exponen function-calling igual.** IAConnect es multi-proveedor por diseño de tenant (`scripts/01_create_database.sql:31-53`); la capa debe normalizar, y **un proveedor sin tool-calling deja el caso sin piso** ([ADR-014](#15-adr-014--fallback-ante-proveedor-llm-caído-degradación-determinística-no-failover)).
- 🟨 **El `grep` de genericidad es una regla de disciplina, no un compilador.** Se degrada sola salvo que esté en CI.

### Estado

🟨 **Propuesto.** Depende de [ADR-011](#12-adr-011--alcance-del-mvp-diagnosticar-la-cadena-eventofunciónfuncionubicaciontarifa) y de [ADR-016](#17-adr-016--️-catálogo-canónico-de-tools-t1t6-resuelve-incoherencia-a). Habilita [ADR-001](#2-adr-001--api-adaptadora-boleteriacoreaiapi-como-capa-de-tools). 🟩 Alineado con [`../GDA-Turnos/04-ADR.md`](../GDA-Turnos/04-ADR.md) ADR-006 — **misma decisión, otro consumidor**.

### Evidencia

| Afirmación | Fuente |
|---|---|
| IAConnect no tiene function-calling (cero `tool_use`/`tool_choice`/`function_call`) | grep exhaustivo sobre la solución IAConnect · [`01-SAD.md`](01-SAD.md) §3.3 (RI-1) |
| RAG léxico TF-IDF, top-5, sin threshold; `VectorEmbedding` siempre `null` | `IAConnect.Application/Services/RAGEngine.cs:34-127` · `KnowledgeService.cs:75` |
| El tenant define proveedor, modelo, temperatura y system prompt | `IAConnect/scripts/01_create_database.sql:31-53` |
| GDA-Turnos toma la misma decisión (tools en el gateway) | [`../GDA-Turnos/04-ADR.md`](../GDA-Turnos/04-ADR.md) ADR-006 |
| DR-8: IAConnect debe permanecer genérico | [`01-SAD.md`](01-SAD.md) §3.1 |

---

## 6. ADR-005 — La regla de publicación se reimplementa en la tool, con test de equivalencia

> ⭐ **Es el ADR más importante del bloque.** De él depende que el asistente y el botón «Publicar» **nunca se contradigan**. Si se contradicen, el caso no falla parcialmente: falla del todo, porque el usuario inexperto — que es toda la audiencia — no tiene forma de saber cuál de los dos miente.

### Contexto

🟩 **La regla real es una sola**: debe existir **al menos una tarifa con `Precio > 0` en una función activa** (fuerza **F3**). Todo lo demás son validaciones de wizard o advertencias.

🟩 **Y no tiene dueño.** Vive en code-behind Blazor, duplicada en dos pantallas:

| Ubicación | Qué valida | Efecto |
|---|---|---|
| `ParametrosEventos.razor.cs:390-405` → modal `:422-436` | Publicar evento pausado sin tarifa con precio | **BLOQUEO** |
| `ParametrosEventosEdit.razor.cs:1090-1105` → `:1165+` | Despausar desde edición sin tarifa con precio | **BLOQUEO** |
| `ParametrosEventosEdit.razor.cs:1019-1034` → `:1149-1163` | Desactivar la última función con precios | **Despublicación automática** |

🟩 **No hay Service ni excepción de dominio que la cubra** (grep exhaustivo sobre `Services/` y `Exceptions/`; las excepciones existentes son todas de compra/carrito/gateway). Fuerza **F4**.

🟩 **Y la propia regla ya se contradice a sí misma, hoy, en producción**: en la **misma pantalla**, `AccionCambiarEstado` (`:386-420`) verifica tarifas y `AccionPausar` (`:441-461`) **despausa sin verificar**. 🟨 Es decir: **existe un camino soportado por la UI que publica un evento que la propia UI considera no publicable.**

🟨 Ese hecho reencuadra todo el ADR. No estamos eligiendo entre "copiar la regla" y "reusar la regla verdadera": **no hay una regla verdadera que reusar.** Hay dos code-behinds que casi coinciden, un tercer camino que no valida nada, y 🟩 sprocs cuyo cuerpo no está en el repo ([ADR-012](#13-adr-012--stored-procedures-no-verificables-se-bloquea-la-capacidad-no-se-adivina)) y que podrían contener reglas invisibles. 🟩 Sin proyecto de tests que fije el comportamiento ([`ia-db ADR-0008`](../../../../../BD/Boleteria.Core.Documentacion/boleteria-core-docs/docs/04-decisions/)).

```mermaid
flowchart TD
    subgraph HOY["🟩 Hoy — la regla no tiene dueño"]
        P1["ParametrosEventos<br/>AccionCambiarEstado :386-420<br/>✅ valida"]
        P2["ParametrosEventos<br/>AccionPausar :441-461<br/>❌ NO valida"]
        P3["ParametrosEventosEdit<br/>:1090-1105<br/>✅ valida"]
        SP["sprocs<br/>❓ cuerpo ausente"]
        P1 --> DB[("Pausado / Activo")]
        P2 --> DB
        P3 --> DB
        SP -.-> DB
    end
    subgraph MAN["🟨 Mañana — con ADR-005"]
        T["DiagnosticoPublicacionService<br/>en AI.Api<br/>4ª implementación"]
        EQ["Test de equivalencia<br/>oráculo vs. LINQ de :394-398"]
        T --- EQ
        EQ -.->|"CI rompe si divergen"| P1
    end
    DB -.-> T

    classDef bad fill:#fce8e6,stroke:#d93025,color:#000
    classDef ok fill:#e6f4ea,stroke:#137333,color:#000
    class P2,SP bad
    class T,EQ ok
```

### Decisión

**La regla se reimplementa en `DiagnosticoPublicacionService`, dentro de `BoleteriaCore.AI.Api`, como consulta de sólo lectura sobre la cadena `Evento→Función→FuncionUbicacion→Tarifa`. Se acepta la duplicación de forma explícita y se la contiene con un test de equivalencia obligatorio en CI que compara la salida del servicio contra el predicado literal del code-behind (`ParametrosEventos.razor.cs:394-398`) sobre un fixture compartido. Si divergen, rompe la build. El servicio es el oráculo del asistente; el code-behind sigue siendo el oráculo del botón.**

🟨 Snippet **propuesto** — el predicado a replicar es el que 🟩 vive en `ParametrosEventos.razor.cs:390-405`:

```csharp
// 🟨 PROPUESTA — BoleteriaCore.AI.Api/Services/DiagnosticoPublicacionService.cs
// ★★ Reimplementa el predicado de ParametrosEventos.razor.cs:394-398.
// ⚠ CUALQUIER cambio acá exige correr EquivalenciaPublicacionTests (ADR-005).
public CausaNoPublicado Diagnosticar(DiagnosticoContexto ctx)
{
    // F2 🟩: "Publicado" no existe en la base. Se deriva de dos flags independientes.
    if (!ctx.Pausado && ctx.Activo && ctx.TieneFuncionActivaConPrecio) return CausaNoPublicado.Ninguna;

    // 🟩 Estado imposible por UI, alcanzable por UpdateByPausado directo (F4).
    if (!ctx.Pausado && !ctx.Activo) return CausaNoPublicado.Inconsistente;

    if (ctx.Funciones.Count == 0)                    return CausaNoPublicado.SinFunciones;
    if (!ctx.Funciones.Any(f => f.Activo))           return CausaNoPublicado.FuncionesInactivas;
    if (!ctx.Funciones.Any(f => f.Ubicaciones.Any())) return CausaNoPublicado.SinUbicaciones;

    // ⭐ F3 🟩: la regla real. El caso del 80%.
    if (!ctx.TieneFuncionActivaConPrecio)            return CausaNoPublicado.TarifasSinPrecio;

    return CausaNoPublicado.Desconocida; // ⚠ jamás se infiere: hand-off (ADR-017)
}
```

🟨 **El test de equivalencia es la decisión, no un accesorio.** Sin él, este ADR es "copiemos la regla y crucemos los dedos":

```csharp
// 🟨 PROPUESTA — EquivalenciaPublicacionTests.cs
[Theory, MemberData(nameof(FixtureDeEventos))]  // 🟩 un caso por valor del enum + los reales
public void ElDiagnosticoCoincideConElBoton(EventoFixture ev)
{
    // Oráculo: el predicado LITERAL del code-behind 🟩 :394-398
    bool puedePublicarSegunLaUI = ev.Funciones.Any(f => f.Activo &&
        f.Ubicaciones.Any(u => u.Tarifas.Any(t => t.Precio > 0)));

    var causa = _svc.Diagnosticar(ev.AContexto());
    Assert.Equal(puedePublicarSegunLaUI, causa is CausaNoPublicado.Ninguna);
}
```

### Alternativas consideradas

| Alternativa | Atractivo | Por qué se descarta |
|---|---|---|
| **Extraer la regla a un Service compartido** que usen la UI **y** la tool | 🟦 **Es la respuesta correcta de manual.** Una sola fuente de verdad; la divergencia se vuelve imposible por construcción; de paso arregla la inconsistencia 🟩 `AccionPausar`/`AccionCambiarEstado`. | 🟩 **Refactor del anfitrión, fuera del alcance del estudio (DR-1).** Tocar `ParametrosEventos.razor.cs` y `ParametrosEventosEdit.razor.cs` — 🟩 el wizard vecino tiene **6.212 líneas** y la pieza suma 11.777 — 🟩 **sin un solo test que respalde el cambio** ([`ia-db ADR-0008`](../../../../../BD/Boleteria.Core.Documentacion/boleteria-core-docs/docs/04-decisions/)). 🟨 **Honestidad brutal**: el riesgo no es técnico sino de gobierno — un bug introducido al refactorizar el flujo de publicación de eventos **rompe la venta de entradas**, y lo habría causado un proyecto de IA que prometía no tocar nada. La relación riesgo/beneficio no la decide este documento. **Se registra como la decisión correcta a futuro** y se propone como candidata explícita de la fase 2 si CE-1 valida el caso ([ADR-015](#16-adr-015--medición-del-éxito-y-criterio-de-continuidad-go--no-go)). |
| **Leer la regla vía un sproc nuevo** (`sp_..._GetBy_EsPublicable`) | Una sola fuente, del lado del dato; la UI podría adoptarlo después; sin refactor de Blazor. | 🟩 **Empeora la observabilidad, que es lo que el caso necesita.** Los cuerpos de los sprocs **no están en el repo** (`DataManager/Migraciones/` tiene 2 archivos) ⇒ la regla quedaría **invisible al code review y no testeable en CI** — justo el problema que [ADR-012](#13-adr-012--stored-procedures-no-verificables-se-bloquea-la-capacidad-no-se-adivina) declara inaceptable. 🟨 Y devolvería un booleano: el caso necesita **qué falta y dónde**, no `false`. 🟨 Un sproc que devuelva el diagnóstico completo es un sproc con `CASE` anidados versionado fuera de Git. ❌ |
| **La tool invoca la UI / reusa el code-behind** | Fuente de verdad literal, cero duplicación. | 🟨 Técnicamente inviable: el predicado está **dentro de un componente Blazor con estado de circuito**, no es invocable desde una API sin instanciar el render tree. ❌ |
| **No diagnosticar: el asistente sólo explica la regla en general** | Cero duplicación, cero riesgo de divergencia. Sólo KB. | 🟨 **Es cancelar el caso.** 🟩 La definición del usuario pide *«indicar ante una pregunta por qué el evento no se publicó qué configuración le faltó»* — sobre **ese** evento. Explicar la regla en abstracto es el modal que ya existe y que el usuario ya no entendió. ❌ |

### Consecuencias positivas

- 🟨 **El diagnóstico es determinista y testeable.** CE-1 (≥95% de acierto) se mide sobre un enum, no sobre prosa.
- 🟨 **La divergencia deja de ser silenciosa.** Hoy nadie se enteraría; con el test de equivalencia, **rompe la build** de quien la introduce.
- 🟩 **El fixture obliga a enumerar los casos reales** — incluido 🟩 el estado `Inconsistente` (`Pausado=false, Activo=false`) que `AccionPausar` puede producir y que ninguna pantalla explica. 🟨 **El asistente va a ser el primer componente del sistema capaz de nombrar ese estado.**
- 🟨 **El diff sobre el anfitrión sigue siendo una línea.** DR-1 intacto.
- 🟨 **Es reversible.** Si mañana se extrae el Service compartido, `DiagnosticoPublicacionService` delega y el test de equivalencia se vuelve el test de la migración.

### Consecuencias negativas

- 🟨 **Es una cuarta implementación de la regla** (dos code-behinds + los sprocs opacos + esta). El documento no lo disfraza: **estamos agregando deuda para no tocar deuda.**
- 🟨 **El test de equivalencia protege el predicado, no la pantalla.** Si alguien cambia `ParametrosEventos.razor.cs:394-398`, el test **compara contra su propia copia del oráculo** y sigue verde. 🟨 Mitigación parcial y honesta: comentario `⚠ ADR-005` en ambos sitios + ítem en el checklist de PR. 🟦 **Es una salvaguarda social, y las salvaguardas sociales fallan.** Riesgo residual **aceptado y registrado**.
- 🟩 **Los sprocs pueden contener reglas que no vemos.** Si `sp_..._UpdateBy_Pausado` valida algo internamente, el diagnóstico puede decir "publicable" y el botón fallar igual. [ADR-012](#13-adr-012--stored-procedures-no-verificables-se-bloquea-la-capacidad-no-se-adivina) acota; no elimina.
- 🟩 **Cobertura incompleta declarada.** 🟨 De `ParametrosEventosAlta.razor.cs` (6.212 líneas) se leyeron ~1.800: **no se leyeron 1508-2719 ni 3440-6212**. 🟨 Y el flujo de **funciones ilimitadas** (`ParametrosEventosAltaFuncionesIlimitadas`, `FechaIlimitadaModel`) no se analizó: **puede tener reglas de publicación propias** ⇒ `Desconocida` va a dispararse más de lo previsto. Es la primera verificación del sprint 0.
- 🟨 **`Pausado` no está mapeada en el Model** ⇒ el servicio la lee cruda del `DataRow` como hace la UI (`ParametrosEventos.razor.cs:194,472`). Deuda heredada, replicada a conciencia.

### Estado

🟨 **Propuesto.** Depende de [ADR-001](#2-adr-001--api-adaptadora-boleteriacoreaiapi-como-capa-de-tools) y [ADR-017](#18-adr-017--️-nomenclatura-canónica-del-enum-causanopublicado-resuelve-incoherencia-b). Habilita [ADR-002](#3-adr-002--deep-links-devueltos-por-la-tool-jamás-construidos-por-el-llm). Limitado por [ADR-012](#13-adr-012--stored-procedures-no-verificables-se-bloquea-la-capacidad-no-se-adivina). 🟨 **Candidato a ser supersedido** por "extraer Service compartido" en fase 2.

### Evidencia

| Afirmación | Fuente |
|---|---|
| La regla real: ≥1 tarifa con `Precio > 0` en función activa | `ParametrosEventos.razor.cs:390-405` → modal `:422-436` |
| Regla duplicada en la pantalla de edición | `ParametrosEventosEdit.razor.cs:1090-1105` → `:1165+` |
| Despublicación automática al desactivar la última función con precios | `ParametrosEventosEdit.razor.cs:1019-1034` → `:1149-1163` |
| ⚠ `AccionPausar` despausa sin verificar tarifas; `AccionCambiarEstado` sí verifica | `ParametrosEventos.razor.cs:441-461` vs. `:386-420` |
| Sin Service ni excepción de dominio que cubra la publicación | grep exhaustivo sobre `Services/` y `Exceptions/` |
| El precio vive en la tabla puente, no en `sys_Tarifas` | `SysTarifasUFuncionUbicacionModel.cs:17-19` · `SysTarifasModel.cs:11-33` |
| `Pausado` no mapeada; se lee cruda del `DataRow` | `ParametrosEventos.razor.cs:194,472` · `SysVentaEntradasEventosModel.cs:57` |
| Sin proyecto de tests en la solución | [`ia-db ADR-0008`](../../../../../BD/Boleteria.Core.Documentacion/boleteria-core-docs/docs/04-decisions/) · [`../Antecedentes/Relevamiento-Verificacion-BoleteriaCore.md:173`](../Antecedentes/Relevamiento-Verificacion-BoleteriaCore.md#no-verificado--límites-de-esta-verdad-de-referencia) |
| Cuerpos de sprocs ausentes del repo | `DataManager/Migraciones/issue-505.sql`, `issue-506.sql` |
| Wizard de 6.212 líneas; cobertura de lectura parcial (no leídas 1508-2719 ni 3440-6212) | `ParametrosEventosAlta.razor.cs` · [`../Antecedentes/Relevamiento-Verificacion-BoleteriaCore.md:175`](../Antecedentes/Relevamiento-Verificacion-BoleteriaCore.md#no-verificado--límites-de-esta-verdad-de-referencia) |
| `DiagnosticoPublicacionService` «reimplementa el LINQ de :394-398» | [`07-Plan-Sprints-Capacitacion.md`](07-Plan-Sprints-Capacitacion.md#298) T-2.3 |

---

## 7. ADR-006 — Arquitectura de conocimiento híbrida: RAG para lo estable, tools para lo volátil

### Contexto

🟩 El RAG de IAConnect es **léxico TF-IDF, top-5, sin threshold de similitud** (`RAGEngine.cs:34-127`); `VectorEmbedding` es `null` siempre (`KnowledgeService.cs:75`). Fuerza **F5**.

🟨 «Sin threshold» es el hecho decisivo y merece nombre propio: **el RAG siempre devuelve cinco fragmentos**, aunque la pregunta no tenga nada que ver con ellos. No existe el resultado vacío. Combinado con 🟩 un `PromptBuilder` que interpola sin escapado (`PromptBuilder.cs:10-55`), el modelo **siempre** recibe contexto que *parece* pertinente. 🟨 Para una pregunta como *«¿por qué no se publicó el evento 42?»*, el RAG va a devolver los cinco fragmentos más parecidos léxicamente a "publicar evento" — y el modelo, obediente, **va a redactar una causa plausible y completamente inventada**.

🟨 Por eso este ADR no es "RAG o tools": es **dónde está prohibido usar RAG**.

### Decisión

**Se adopta una arquitectura híbrida con una frontera explícita y no negociable: todo lo que dependa del estado de una fila se responde con tools; todo lo que sea estable, conceptual o procedimental se responde con RAG. Ningún dato de un evento concreto entra jamás a la KB. Ante una pregunta sobre un evento puntual sin tool disponible, el asistente deriva; no responde con RAG.**

| Tipo de consulta | Ejemplo | Mecanismo | Por qué | Marca |
|---|---|---|---|---|
| Estado de un evento concreto | *«¿por qué no se publicó el 42?»* | **Tool T1** | 🟩 Depende de filas. Ningún texto lo contiene | 🟩 |
| Listado del alcance del usuario | *«¿qué eventos tengo?»* | **Tool T2** | 🟩 `Publicado` no existe: hay que derivarlo de `!Pausado` | 🟩 |
| Precios de una función | *«¿la platea tiene precio?»* | **Tools T4+T5** | 🟩 Precio vive en la tabla puente | 🟩 |
| Valores de lookup | *«¿qué tipos de evento hay?»* | **Tool T6** | 🟨 Cambian sin deploy; en la KB se pudren | 🟨 |
| Concepto del modelo | *«¿qué es una FuncionUbicacion?»* | **RAG** | 🟨 Estable; no depende de datos | 🟨 |
| Regla de negocio en abstracto | *«¿qué necesita un evento para publicarse?»* | **RAG** | 🟨 Estable. 🟩 La regla es una sola (F3) | 🟨 |
| Procedimiento | *«¿cómo cargo una tarifa?»* | **RAG** | 🟨 Guía paso a paso; no depende del evento | 🟨 |
| Mapa de pantallas | *«¿dónde configuro los lugares?»* | **RAG** + link sin parámetros | 🟨 Rutas sin `{id}` son texto estable | 🟨 |
| Ventas / recaudación | *«¿cuánto vendí?»* | ⚠ **Fuera del MVP** | [ADR-011](#12-adr-011--alcance-del-mvp-diagnosticar-la-cadena-eventofunciónfuncionubicaciontarifa) | 🟨 |
| Vigencia | *«¿está vigente?»* | ⚠ **Bloqueada** | 🟩 Sprocs opacos ([ADR-012](#13-adr-012--stored-procedures-no-verificables-se-bloquea-la-capacidad-no-se-adivina)) | 🟩 |

```mermaid
flowchart TD
    Q["Pregunta del usuario"] --> D{"¿La respuesta cambia<br/>si cambia una fila?"}
    D -->|"SÍ"| T{"¿Hay tool<br/>en el catálogo?"}
    D -->|"NO"| R["RAG · KB curada<br/>ADR-013"]
    T -->|"sí"| TOOL["Tool T1…T6<br/>ADR-016"]
    T -->|"no"| H["⚠ HAND-OFF<br/>Mesa de Ayuda"]
    R --> LLM["LLM redacta"]
    TOOL --> LLM
    H -.->|"❌ jamás<br/>rellenar con RAG"| R

    classDef bad fill:#fce8e6,stroke:#d93025,stroke-width:2px,color:#000
    classDef ok fill:#e6f4ea,stroke:#137333,color:#000
    class H bad
    class TOOL,R ok
```

🟨 **La flecha punteada roja es el ADR entero.** El modo de falla que mata el caso no es "no sé": es **rellenar con RAG una pregunta que era de tool**. Y como el RAG nunca devuelve vacío, ese relleno **siempre está disponible**.

### Alternativas consideradas

| Alternativa | Atractivo | Por qué se descarta |
|---|---|---|
| **Todo RAG** | IAConnect ya lo tiene. Cero desarrollo. Semanas de diferencia. | 🟩 No puede responder la pregunta del caso. 🟨 Y como el TF-IDF sin threshold nunca dice "no sé", el fracaso sería **invisible**: respuestas fluidas, seguras y falsas. ❌ |
| **Todo tools, sin KB** | Máximo determinismo. Nada que curar. | 🟨 Necesitaría una tool por concepto (*«¿qué es una FuncionUbicacion?»*) — código nuevo para responder algo que un párrafo responde mejor. 🟨 Y sin KB, el system prompt cargaría todo el vocabulario del dominio. ❌ |
| **RAG vectorial** en vez de TF-IDF | 🟦 Mejor recall; resolvería sinónimos (*«función»* vs. *«fecha»* vs. *«presentación»*). | 🟩 `SerializeEmbedding` existe pero es **código muerto**; habría que construir el pipeline y elegir proveedor de embeddings. 🟨 Fuera del alcance de este bloque: es una decisión **del gateway**, no del caso. Se registra como candidata para [`../Ng-IAServices/04-ADR.md`](../Ng-IAServices/04-ADR.md). 🟨 Mitigación provisoria: [ADR-013](#14-adr-013--curaduría-y-propiedad-de-la-kb-dueño-funcional--pipeline-idempotente) escribe la KB **léxico-first**, con sinónimos incrustados en el texto. |
| **Poner el estado de los eventos en la KB** y refrescarlo periódicamente | Sin function-calling. El RAG "sabría" de eventos. | 🟨 **Es la peor opción del documento.** Datos rancios presentados con seguridad: el usuario cargó el precio hace 2 minutos y el asistente insiste en que falta. 🟦 Y viola el aislamiento: la KB es por tenant, los eventos son por usuario ⇒ el organizador A leería los eventos de B ([ADR-003](#4-adr-003--propagación-de-identidad-token-exchange-de-la-cookie-del-backoffice)). ❌❌ |

### Consecuencias positivas

- 🟨 **Cada mecanismo hace lo que sabe hacer.** El determinismo donde hay que acertar; el lenguaje donde hay que explicar.
- 🟨 **La KB no tiene datos personales ni de negocio** ⇒ su exposición no es un incidente de datos.
- 🟨 **La KB no se pudre.** Sólo contiene lo estable; 🟩 los lookups los trae T6 (`06-Administrator-Guide.md:823` ya lo instruye: *«No listes los tipos de evento en la KB»*).
- 🟦 **La frontera es auditable**: toda respuesta sobre un evento concreto debe tener una invocación de tool en el log. Si no la tiene, es alucinación — y se detecta.

### Consecuencias negativas

- 🟨 **Dos sistemas de conocimiento que mantener**, con dueños distintos ([ADR-013](#14-adr-013--curaduría-y-propiedad-de-la-kb-dueño-funcional--pipeline-idempotente)).
- 🟨 **La frontera la respeta el modelo, y el modelo es estocástico.** El system prompt puede decir "no respondas sobre eventos concretos sin tool" y el modelo hacerlo igual. 🟦 Mitigación: guardrail de salida + CE-8 medido sobre muestra. **Riesgo residual real.**
- 🟩 **TF-IDF castiga el vocabulario del usuario inexperto**, que es justo la audiencia: si pregunta *«¿por qué no sale mi obra?»* y la KB dice *«evento»*, la similitud léxica es baja y el top-5 trae ruido. Sin threshold, el ruido llega igual al prompt.
- 🟨 **T6 agrega una tool para algo que "podría" ser KB.** Es deliberado: 🟩 los lookups cambian sin deploy.

### Estado

🟨 **Propuesto.** Depende de [ADR-011](#12-adr-011--alcance-del-mvp-diagnosticar-la-cadena-eventofunciónfuncionubicaciontarifa) y [ADR-016](#17-adr-016--️-catálogo-canónico-de-tools-t1t6-resuelve-incoherencia-a). Habilita [ADR-013](#14-adr-013--curaduría-y-propiedad-de-la-kb-dueño-funcional--pipeline-idempotente). 🟩 Alineado con [`../GDA-Turnos/04-ADR.md`](../GDA-Turnos/04-ADR.md) ADR-004 (*«RAG para lo estable, tools para lo volátil»*) — **mismo principio, otro dominio**.

### Evidencia

| Afirmación | Fuente |
|---|---|
| RAG léxico TF-IDF top-5 **sin threshold** | `IAConnect.Application/Services/RAGEngine.cs:34-127` |
| `VectorEmbedding` siempre `null`; `SerializeEmbedding` es código muerto | `IAConnect.Application/Services/KnowledgeService.cs:75` |
| `PromptBuilder` interpola sin escapado | `IAConnect.Application/Services/PromptBuilder.cs:10-55` |
| `Publicado` no existe: se deriva de `!Pausado` | `ParametrosEventosEdit.razor.cs:174` |
| La instrucción de no listar lookups en la KB ya está escrita | [`06-Administrator-Guide.md:823`](06-Administrator-Guide.md) |
| Vigencia bloqueada por sprocs opacos | [`02-HLD.md:1580-1584`](02-HLD.md) §12.1 |
| Principio híbrido en el bloque hermano | [`../GDA-Turnos/04-ADR.md`](../GDA-Turnos/04-ADR.md) ADR-004 |

---

## 8. ADR-007 — El asistente no ejecuta acciones: tools de sólo lectura en la v1

### Contexto

🟨 La pregunta es inevitable y la va a hacer el primer usuario de la demo: si el asistente **sabe** que falta el precio y **sabe** dónde cargarlo, ¿por qué no lo carga? Y si el evento **cumple** la regla y está pausado por olvido, ¿por qué no lo publica?

🟩 Técnicamente sería fácil: `UpdateByPausado` existe y es invocable (`SysVentaEntradasEventosDataManager.cs:32-42`). 🟩 **Demasiado fácil**: es precisamente el método que `AccionPausar` (`:441-461`) llama **sin validar tarifas**, produciendo el estado inconsistente. 🟨 Es decir: la ruta de escritura más accesible del dominio es también la **menos protegida**.

🟩 Publicar un evento es un acto con consecuencias externas e irreversibles: lo expone en el portal público de compra (`BoleteriaCore.Web`). 🟨 No hay «despublicar y que nadie se haya enterado»: si alguien compró, hay entradas emitidas, cobros por gateway y un problema que no se resuelve con un `UPDATE`.

🟩 Y no hay red: sin tests, sin transaccionalidad verificable, con los sprocs opacos, con la validación viviendo client-side en un code-behind que la propia UI se saltea por un camino.

### Decisión

**Todas las tools de la v1 son de sólo lectura. El asistente diagnostica, explica y **deriva al flujo nativo** mediante deep-link. La escritura la ejecuta el usuario, en la pantalla del Backoffice, con el botón que ya existe y las validaciones que ya corren. `BoleteriaCore.AI.Api` expone **exclusivamente** operaciones `SELECT`; el usuario de base de datos de la API tiene permisos de sólo lectura, y eso se verifica en el despliegue.**

🟨 La barrera es **de infraestructura, no de disciplina**: aunque alguien programara una tool de escritura por error, el usuario de BD la rechaza. 🟦 Defensa en profundidad: la política no depende de que nadie se equivoque.

```mermaid
stateDiagram-v2
    [*] --> Pregunta
    Pregunta --> Diagnostico: T1 (SELECT)
    Diagnostico --> Explicacion: causa + eslabón
    Explicacion --> DeepLink: ADR-002
    DeepLink --> PantallaNativa: el USUARIO navega
    PantallaNativa --> BotonPublicar: validaciones 🟩 :390-405
    BotonPublicar --> [*]: publicado por una persona

    Diagnostico --> Escritura: ❌ PROHIBIDO v1
    note right of Escritura
        🟩 UpdateByPausado existe
        🟩 y AccionPausar lo llama
        sin validar (F4)
        ⇒ barrera: permisos de BD
    end note
```

### Alternativas consideradas

| Alternativa | Atractivo | Por qué se descarta |
|---|---|---|
| **El asistente publica con confirmación explícita** (*«¿publico?» → «sí»*) | 🟦 Patrón de industria maduro (human-in-the-loop). Cierra el bucle: el usuario resuelve sin salir del chat. | 🟨 **La confirmación no es consentimiento informado cuando el usuario es inexperto** — que es toda la audiencia. Confirma lo que el asistente le dijo, no lo que el sistema va a hacer. 🟩 Y hay un agujero concreto: si la tool escribiera vía `UpdateByPausado` **replicaría la falla de `AccionPausar`** (`:441-461`), que despausa sin validar. 🟨 Estaríamos automatizando el camino roto. Diferido a v2, y **sólo** si CE-1 ≥ 95% sostenido ([ADR-015](#16-adr-015--medición-del-éxito-y-criterio-de-continuidad-go--no-go)). |
| **Escrituras de bajo riesgo** (cargar un precio) sí; publicar no | Resuelve el 80% 🟩 (`TarifasSinPrecio`) sin tocar lo irreversible. | 🟨 **«Bajo riesgo» es falso**: 🟩 cargar un precio en una función activa **publica el evento de hecho**, porque es la condición de la regla F3. La escritura "inocente" tiene el efecto de la peligrosa. 🟨 Que la distinción sea *contraintuitiva* es la mejor prueba de que el asistente no debe escribir todavía. ❌ |
| **El asistente prepara el formulario** (pre-llena y el usuario guarda) | Buen balance: el humano ejecuta. | 🟩 Exige tocar el wizard — 6.212 líneas, sin tests. Rompe DR-1. ❌ |
| **Sólo lectura para siempre** | Máxima seguridad; cero ambigüedad. | 🟨 Cierra la puerta antes de tener datos. 🟨 Si el diagnóstico demuestra ≥95% de acierto durante un trimestre, la conversación cambia. Se prefiere **diferir** a **prohibir**. |

### Consecuencias positivas

- 🟨 **El peor caso del asistente es decir algo equivocado, no hacer algo equivocado.** Un diagnóstico errado cuesta confusión; una publicación errada cuesta plata y entradas vendidas.
- 🟨 **El usuario aprende dónde está la pantalla.** 🟩 El caso es explícitamente pedagógico: *«que sirva de guía para usuarios inexpertos»*. Un asistente que ejecuta no enseña: reemplaza.
- 🟨 **Las validaciones nativas siguen corriendo**, incluidas las que no conocemos (sprocs, [ADR-012](#13-adr-012--stored-procedures-no-verificables-se-bloquea-la-capacidad-no-se-adivina)).
- 🟦 **Prompt injection deja de ser un vector de escritura.** Con sólo lectura, el peor caso de una KB envenenada es texto; con escritura, sería un `UPDATE`.
- 🟨 **Achica el alcance de auditoría y de aprobación**: un asistente que no escribe es mucho más fácil de aprobar.

### Consecuencias negativas

- 🟨 **El usuario tiene que hacer el trabajo.** El asistente lo lleva a la puerta; no entra por él. Menos "mágico" en la demo.
- 🟨 **Se pierde la oportunidad de arreglar `AccionPausar` de paso.** Una tool de publicación bien hecha sería **más correcta que el botón**. 🟨 Paradoja registrada: no la tomamos porque el sistema no está listo para que la tomemos.
- 🟨 **Presión de producto garantizada.** «Ya que sabe, que lo haga» va a volver en cada demo. Este ADR es la respuesta escrita, con condición de revisión explícita.
- 🟨 **Los permisos de sólo lectura hay que crearlos y sostenerlos.** 🟩 `DataEntityCore.Configure()` es un singleton estático con **una** connection string: la API adaptadora necesita la suya, distinta de la del host, y nadie debe "arreglarla" reusando la del BO.

### Estado

🟨 **Propuesto**, con revisión condicionada a [ADR-015](#16-adr-015--medición-del-éxito-y-criterio-de-continuidad-go--no-go). Depende de [ADR-011](#12-adr-011--alcance-del-mvp-diagnosticar-la-cadena-eventofunciónfuncionubicaciontarifa) y [ADR-003](#4-adr-003--propagación-de-identidad-token-exchange-de-la-cookie-del-backoffice). Habilita [ADR-002](#3-adr-002--deep-links-devueltos-por-la-tool-jamás-construidos-por-el-llm).

### Evidencia

| Afirmación | Fuente |
|---|---|
| `UpdateByPausado` existe y es invocable directamente | `SysVentaEntradasEventosDataManager.cs:32-42` |
| `AccionPausar` despausa sin validar tarifas | `ParametrosEventos.razor.cs:441-461` |
| Las validaciones de publicación son client-side, sin Service | `ParametrosEventos.razor.cs:390-405` · grep sobre `Services/`, `Exceptions/` |
| Publicar expone el evento en el portal público | [`ia-db/indexes/04_Portal-Publico.md`](../../../../../BD/Boleteria.Core.Documentacion/ia-db/indexes/04_Portal-Publico.md) |
| `DataEntityCore.Configure()` singleton con una sola connection string | `Notions.Core.Utils.DataManager/DataEntityCore.cs:33-256` |
| Sin proyecto de tests | [`ia-db ADR-0008`](../../../../../BD/Boleteria.Core.Documentacion/boleteria-core-docs/docs/04-decisions/) |
| El caso es pedagógico («guía para usuarios inexpertos») | Definición textual del usuario · [`01-SAD.md`](01-SAD.md) §2.1 |

---

## 9. ADR-008 — Widget como componente Blazor en `MainLayout`, no script de CDN

### Contexto

🟩 El Backoffice es Blazor con `MainLayout.razor` como layout raíz y un sidebar cuyos ítems se renderizan condicionalmente por perfil (`MainLayout.razor.cs:79`, `MainLayout.razor:54`). 🟩 Los hosts son net10 (`01-SAD.md` RA-1).

🟨 El widget necesita tres cosas que un `<script>` genérico no tiene: (1) la **identidad** del usuario para el token-exchange ([ADR-003](#4-adr-003--propagación-de-identidad-token-exchange-de-la-cookie-del-backoffice)); (2) `NavigationManager` para resolver el **PathBase** de los deep-links (🟩 las rutas del BO se sirven bajo un PathBase obligatorio, [`routes-map.md`](../../../../../BD/Boleteria.Core.Documentacion/boleteria-core-docs/docs/pieces/boleteria-core-backoffice/routes-map.md)); (3) el **contexto de pantalla** — saber que el usuario está en `/ParametrosEventosEdit?idEvento=42` permite abrir el chat sabiendo de qué evento habla.

🟩 Y el driver duro sigue siendo DR-1: no reescribir el anfitrión.

### Decisión

**El widget es un componente Blazor (`<AsistenteWidget />`) declarado una vez en `MainLayout.razor`, dentro del `@if` de perfil que ya gobierna el menú. El diff sobre el código existente es de una línea. El widget resuelve el PathBase con `NavigationManager` y mantiene la allowlist de rutas como segunda barrera de [ADR-002](#3-adr-002--deep-links-devueltos-por-la-tool-jamás-construidos-por-el-llm).**

### Alternativas consideradas

| Alternativa | Atractivo | Por qué se descarta |
|---|---|---|
| **`<script>` de CDN** (widget JS embebible) | 🟦 Patrón clásico; reusable por cualquier host, incluso no-Blazor; desacoplado del anfitrión. | 🟨 No tiene identidad ni `NavigationManager`: el token-exchange y el PathBase habría que pasárselos por atributos, reinventando la integración. 🟨 Y el estilo divergiría del Backoffice, que es justo la señal de confianza que el caso necesita. 🟨 **Se registra como la opción correcta para `BoleteriaCore.Web`** (portal público) si el caso se extiende. |
| **`<iframe>`** | Aislamiento total; cero conflicto de CSS. | 🟨 El aislamiento es el problema: sin acceso a `NavigationManager`, los deep-links tendrían que salir con `target="_parent"` y el PathBase se resuelve a mano. 🟨 Y un iframe no puede leer la cookie del circuito. ❌ |
| **Página dedicada `/asistente`** | Simplísimo. Sin tocar `MainLayout`. | 🟨 **Rompe el caso de uso.** El usuario está mirando el modal *«No se puede publicar el evento»* 🟩 (`:422-436`): pedirle que navegue a otra pantalla, pierda el contexto y vuelva a explicar cuál era su evento es exactamente la fricción que el asistente vino a eliminar. ❌ |
| **Integrarlo en el modal de error** (contextual, sin chat) | Máxima relevancia: aparece justo donde falla. | 🟨 Exige tocar el modal 🟩 y ese modal vive en los code-behinds de 6.212 y 11.777 líneas. Rompe DR-1. 🟨 **Candidato de fase 2** — es la mejor UX del documento, y hoy no la podemos pagar. |

### Consecuencias positivas

- 🟨 **DR-1 verificable**: una línea de diff, medible en el PR.
- 🟨 **La visibilidad se gobierna con el mecanismo que ya existe**: 🟩 el mismo `TienePerfil()` del sidebar decide quién ve el asistente. Sin autorización nueva en el cliente.
- 🟨 **El contexto de pantalla es gratis.** El widget sabe el `idEvento` de la query string y puede pre-cargar el diagnóstico.
- 🟨 **El estilo es el del Backoffice** porque *es* el Backoffice.

### Consecuencias negativas

- 🟨 **Acopla el widget a Blazor.** No es reusable en `BoleteriaCore.Web` ni en un host futuro no-Blazor. Deuda aceptada: el MVP es sólo Backoffice ([ADR-011](#12-adr-011--alcance-del-mvp-diagnosticar-la-cadena-eventofunciónfuncionubicaciontarifa)).
- 🟨 **Vive en el circuito.** Un chat abierto mantiene estado de servidor en todas las páginas; con muchos usuarios concurrentes es consumo real de memoria. **No verificado**: la carga concurrente del BO.
- 🟨 **Está en todas las pantallas**, incluidas las que no tienen nada que ver con eventos. El system prompt debe acotar el alcance o el usuario preguntará por cajeros y puntos de venta.
- 🟩 **PathBase no verificado por ambiente.** El widget lo resuelve en runtime, pero nadie confirmó su valor en producción.

### Estado

🟨 **Propuesto.** Depende de [ADR-011](#12-adr-011--alcance-del-mvp-diagnosticar-la-cadena-eventofunciónfuncionubicaciontarifa). Habilita [ADR-002](#3-adr-002--deep-links-devueltos-por-la-tool-jamás-construidos-por-el-llm) y [ADR-003](#4-adr-003--propagación-de-identidad-token-exchange-de-la-cookie-del-backoffice).

### Evidencia

| Afirmación | Fuente |
|---|---|
| `MainLayout` es el layout raíz; ítems condicionados por perfil | `MainLayout.razor:54` · `MainLayout.razor.cs:79` |
| Las rutas del BO se sirven bajo PathBase obligatorio | [`routes-map.md`](../../../../../BD/Boleteria.Core.Documentacion/boleteria-core-docs/docs/pieces/boleteria-core-backoffice/routes-map.md) |
| Modal «No se puede publicar el evento» | `ParametrosEventos.razor.cs:422-436` |
| Hosts en net10; pieza de 11.777 líneas (RA-1, DR-1) | [`01-SAD.md`](01-SAD.md) §3.1, §3.2 |
| Ítem «Mesa de Ayuda» existente en el sidebar (hand-off) | `MainLayout.razor:54` · [`02-HLD.md:1197`](02-HLD.md) |

---

## 10. ADR-009 — Dos tenants por perfil de usuario, no un system prompt condicional

### Contexto

🟩 `lut_Tenants` es la raíz del particionado de IAConnect: proveedor, modelo, temperatura y **system prompt** son columnas del tenant (`scripts/01_create_database.sql:31-53`). 🟨 Un tenant es, entonces, **una personalidad + una KB + un catálogo de tools**.

🟩 El Backoffice tiene perfiles heterogéneos (`parametros`, `hacienda`, `control-acceso`; `MainLayout.razor.cs:79`). 🟨 Un organizador que carga eventos y un administrador que audita el sistema **no necesitan el mismo asistente**: el primero necesita acompañamiento paso a paso; el segundo, respuestas densas y sin didáctica.

🟩 El HLD ya diseñó **dos narrativas sobre el mismo contrato de tool** (`02-HLD.md:1595-1596`: *«Narrativa organizador / system prompt A»* y *«Narrativa admin / system prompt B»*). 🟨 Y hay tools que **sólo** tienen sentido para admin: 🟩 `explicar_estado_inconsistente` y `listar_eventos_no_publicados` están marcadas «Admin» en el catálogo (`02-HLD.md:1575-1576`).

### Decisión

**Se crean dos tenants de IAConnect, uno por perfil: `boleteria-backoffice-organizador` y `boleteria-backoffice-admin`. Cada uno tiene su system prompt, su KB y su subconjunto del catálogo de tools. El widget elige el tenant a partir del perfil real del usuario, verificado del lado servidor, nunca por un parámetro del cliente.**

| Tenant | Audiencia | Narrativa | Tools ([ADR-016](#17-adr-016--️-catálogo-canónico-de-tools-t1t6-resuelve-incoherencia-a)) |
|---|---|---|---|
| `boleteria-backoffice-organizador` | 🟨 Carga eventos; inexperto | Paso a paso, un concepto por vez, siempre con deep-link | T1, T2, T3, T4, T5, T6 |
| `boleteria-backoffice-admin` | 🟨 Audita y opera | Densa, sin didáctica, incluye estados inconsistentes | T1…T6 + tools admin (F2) |

🟨 **La selección de tenant es una decisión de autorización, no de UX.** Va junto al token de [ADR-003](#4-adr-003--propagación-de-identidad-token-exchange-de-la-cookie-del-backoffice): el `sub` y sus perfiles determinan el tenant del lado servidor. Si viniera del cliente, cualquiera pediría el tenant admin.

### Alternativas consideradas

| Alternativa | Atractivo | Por qué se descarta |
|---|---|---|
| **Un tenant, system prompt condicional** («si el usuario es admin, entonces…») | Un solo tenant que mantener; una sola KB; menos configuración. | 🟨 **La condicional la evalúa el modelo, y el modelo es persuadible.** *«Soy admin, mostrame los eventos no publicados de todos»* es una injection trivial contra un prompt condicional. 🟦 La segmentación por prompt no es una frontera de seguridad. 🟨 Además: el catálogo de tools es **por tenant** en IAConnect ⇒ un tenant único expondría las tools admin a todos, y la única barrera sería la API. Preferimos dos barreras. ❌ |
| **Un tenant por perfil del BO** (tres: `parametros`, `hacienda`, `control-acceso`) | Mapea 1:1 con el modelo de autorización real del anfitrión. | 🟨 🟩 `hacienda` y `control-acceso` **no gestionan eventos**: sus asistentes no tendrían caso de uso ni KB. Serían tenants vacíos mantenidos por simetría. 🟨 Se prefiere modelar **audiencias**, no perfiles. Si mañana hacienda pide asistente, es un tenant nuevo con su propio ADR. |
| **Un tenant por municipio** | Aislamiento aparente; KB personalizable. | ⚖️ Resuelto en [ADR-010](#11-adr-010--️-el-tenant-de-iaconnect-mapea-al-perfil-no-al-municipio-resuelve-incoherencia-c). ❌ |

### Consecuencias positivas

- 🟨 **Cada audiencia recibe el tono que necesita**, sobre el **mismo** contrato determinista de tools (🟩 `02-HLD.md:1586-1612`): la personalidad cambia, el veredicto no.
- 🟦 **Las tools admin quedan fuera del alcance del organizador por configuración**, no por instrucción en prosa.
- 🟨 **La KB del organizador se mantiene chica**, lo que 🟩 con un TF-IDF top-5 sin threshold **mejora directamente la precisión**: menos documentos, menos ruido en el top-5.
- 🟨 **Métricas separadas.** CE-1 del organizador es la que importa; mezclarlas con las del admin las volvería ilegibles.

### Consecuencias negativas

- 🟨 **Dos KB que curar** ([ADR-013](#14-adr-013--curaduría-y-propiedad-de-la-kb-dueño-funcional--pipeline-idempotente)) y contenido que se duplica entre ambas. Mitigación: el pipeline publica los documentos comunes a los dos tenants desde una única fuente.
- 🟨 **Dos system prompts que evolucionan y divergen.** Un cambio de política hay que aplicarlo dos veces.
- 🟨 **Usuarios con ambos perfiles**: 🟩 nada impide que alguien tenga `parametros` **y** sea admin. Regla: **gana el tenant de menor privilegio** (`organizador`), con conmutador explícito en el widget. La ambigüedad se resuelve conservadora.
- 🟨 **Duplica el trabajo de evaluación**: cada cambio de prompt hay que probarlo en dos tenants.

### Estado

🟨 **Propuesto.** Depende de [ADR-011](#12-adr-011--alcance-del-mvp-diagnosticar-la-cadena-eventofunciónfuncionubicaciontarifa). Habilita [ADR-010](#11-adr-010--️-el-tenant-de-iaconnect-mapea-al-perfil-no-al-municipio-resuelve-incoherencia-c). 🟩 Alineado con [`../GDA-Turnos/04-ADR.md`](../GDA-Turnos/04-ADR.md) ADR-002 (`gda-turnos-ciudadano` / `gda-turnos-funcionario`) — **mismo patrón**.

### Evidencia

| Afirmación | Fuente |
|---|---|
| `lut_Tenants`: proveedor, modelo, temperatura y system prompt por tenant | `IAConnect/scripts/01_create_database.sql:31-53` |
| Perfiles del BO: `parametros`, `hacienda`, `control-acceso` | `MainLayout.razor.cs:79` · [`ia-db/indexes/08_Seguridad.md`](../../../../../BD/Boleteria.Core.Documentacion/ia-db/indexes/08_Seguridad.md) |
| Dos narrativas sobre el mismo contrato de tool | [`02-HLD.md:1595-1596`](02-HLD.md) §12.2 |
| Tools marcadas «Admin» en el catálogo | [`02-HLD.md:1575-1576`](02-HLD.md) §12.1 |
| Patrón de dos tenants por perfil en el bloque hermano | [`../GDA-Turnos/04-ADR.md`](../GDA-Turnos/04-ADR.md) ADR-002 |

---

## 11. ADR-010 — ⚖️ El tenant de IAConnect mapea al perfil, no al municipio (resuelve incoherencia **C**)

> ⚖️ **ADR de desempate.** 🟩 `01-SAD.md` §6.6 declara los tenants `boleteria-backoffice-{municipio}` y `boleteria-web-{municipio}`, con el sufijo justificado en §10.3 (`01-SAD.md:718-721`). **Este ADR lo contradice y gana.** El sufijo se elimina.

### Contexto

🟩 **BoleteriaCore no es multi-tenant.** No existe discriminador de aislamiento. Lo más cercano: `GP_IdMunicipio` (`SysVentaEntradasEventosModel.cs:23`) y el parámetro global `CONFIG_codMunicipio`. 🟨 La verdad de referencia es explícita sobre el límite: *«la segmentación parece ser por municipio, pero no hay código que lo confirme como aislamiento»*.

🟩 Y `lut_Parametros` — donde vive `CONFIG_codMunicipio` — es **clave-valor global**: sólo `Codigo`, `Valor`, `Observaciones` (`LutParametrosModel.cs:11-15`). **Sin `Id_Evento`, sin tenant, sin scope.** 🟨 Es decir: `CONFIG_codMunicipio` no es *«el municipio de este usuario»*; es **una constante de la instalación**. Una instalación, un municipio.

🟨 Ese hecho desarma el sufijo. `boleteria-backoffice-{municipio}` sugiere que un IAConnect atiende a varios municipios de un mismo BoleteriaCore — y **eso no es lo que el sistema es**. Si cada municipio tiene su instalación, el sufijo codifica en el nombre del tenant algo que ya está determinado por *qué instancia te está hablando*.

🟨 Hay que decidir qué **es** un tenant de IAConnect. Los hechos 🟩 dicen que es una fila de `lut_Tenants` con proveedor, modelo, temperatura y system prompt (`scripts/01_create_database.sql:31-53`) — es decir: **una personalidad + una KB + un catálogo de tools**. **No es una frontera de datos.** La frontera de datos la impone la API adaptadora con el token ([ADR-003](#4-adr-003--propagación-de-identidad-token-exchange-de-la-cookie-del-backoffice)).

### Decisión

**El identificador de tenant es `boleteria-backoffice-organizador` y `boleteria-backoffice-admin`. Sin sufijo de municipio. El tenant modela la audiencia (perfil), no la instalación ni la jurisdicción. El aislamiento de datos entre municipios lo garantiza exclusivamente `alcance(sub)` en `BoleteriaCore.AI.Api` a partir del JWT; jamás el nombre del tenant. Si una instalación futura sirviera a varios municipios con KB genuinamente distintas, este ADR se supersede — no se parchea con un sufijo.**

| Concepto | Quién lo resuelve | Marca |
|---|---|---|
| Personalidad y tono | Tenant (`System_Prompt`) | 🟩 |
| Qué documentos ve el RAG | Tenant (KB) | 🟩 |
| Qué tools puede invocar | Tenant (`ToolRegistry`, [ADR-004](#5-adr-004--function-calling-genérico-en-iaconnect-no-un-hack-de-boletería)) | 🟨 |
| **Qué eventos puede leer** | **`alcance(sub)` en la AI.Api** — nunca el tenant | 🟨 |
| Qué municipio es esta instalación | 🟩 `CONFIG_codMunicipio`, constante de instalación | 🟩 |

```mermaid
flowchart TD
    subgraph MAL["❌ Descartado — el sufijo miente"]
        M1["boleteria-backoffice-rosario"]
        M2["boleteria-backoffice-santafe"]
        M1 -.->|"KB idéntica<br/>duplicada"| M2
        M1 --> X["🟨 sugiere aislamiento<br/>que el tenant NO da"]
    end
    subgraph BIEN["✅ ADR-010"]
        T1["boleteria-backoffice-organizador"]
        T2["boleteria-backoffice-admin"]
        T1 --> API["AI.Api"]
        T2 --> API
        API --> AL{"alcance(sub)<br/>del JWT"}
        AL --> DB[("eventos del<br/>municipio real")]
    end
    classDef bad fill:#fce8e6,stroke:#d93025,color:#000
    classDef ok fill:#e6f4ea,stroke:#137333,color:#000
    class M1,M2,X bad
    class T1,T2,API,AL,DB ok
```

### Alternativas consideradas

| Alternativa | Atractivo | Por qué se descarta |
|---|---|---|
| **Sufijo `-{municipio}` vía `GP_IdMunicipio`** (lo que dice 🟩 `01-SAD.md:718`) | Aparenta aislamiento. Permitiría KB por municipio. Alineado con el campo que existe. | 🟨 **Aparenta un aislamiento que no da.** El tenant no filtra filas: si la API no valida el alcance, `boleteria-backoffice-rosario` lee los eventos de Santa Fe igual. 🟦 Un nombre que sugiere seguridad inexistente es **peor que ningún nombre**: invita a saltear la validación real. 🟨 Y multiplica el costo de [ADR-013](#14-adr-013--curaduría-y-propiedad-de-la-kb-dueño-funcional--pipeline-idempotente) por N: N KB idénticas que divergen con el primer parche urgente. 🟩 Además `CONFIG_codMunicipio` es global (`LutParametrosModel.cs:11-15`) ⇒ una instalación ya **es** un municipio: el sufijo es redundante. ❌ |
| **Tenant único `boleteria-backoffice`**, sin perfil ni municipio | Lo más simple. Una KB, un prompt. | 🟨 Contradice [ADR-009](#10-adr-009--dos-tenants-por-perfil-de-usuario-no-un-system-prompt-condicional): perdería la segmentación de tools admin y las dos narrativas 🟩 que el HLD ya diseñó (`02-HLD.md:1595-1596`). ❌ |
| **Sufijo por instalación** (`-prod`, `-qa`) | Separa ambientes. | 🟨 Los ambientes ya se separan por **despliegue** de IAConnect, no por fila de `lut_Tenants`. Mezclar ambiente e identidad de negocio en la misma clave es un error conocido. ❌ |
| **Sufijo de municipio sólo si aparece un multi-municipio real** | Difiere la decisión hasta tener el dato. | 🟨 Es, de hecho, **lo que este ADR hace** — declarándolo como cláusula de supersesión explícita en vez de dejar el sufijo "por las dudas". La diferencia es que hoy el nombre no miente. |

### Consecuencias positivas

- 🟨 **El nombre del tenant deja de mentir.** Nadie va a suponer que `-rosario` aísla algo. La única frontera está donde se puede auditar: 🟩 la API que conoce `GP_IdMunicipio`.
- 🟨 **Una sola KB por audiencia.** [ADR-013](#14-adr-013--curaduría-y-propiedad-de-la-kb-dueño-funcional--pipeline-idempotente) es viable con un dueño funcional; con N municipios no lo sería.
- 🟨 **Los tenants dejan de depender de un hecho no verificado.** 🟩 Que `GP_IdMunicipio` sea el criterio de segmentación es 🟨 **inferencia**, no hecho. Construir los identificadores de tenant sobre una inferencia significa renombrarlos si la inferencia cae.
- 🟨 **Las métricas agregan.** CE-1 sobre `boleteria-backoffice-organizador` es una muestra; repartida en N municipios, ninguna alcanza significancia.
- 🟨 **Coherente con [ADR-009](#10-adr-009--dos-tenants-por-perfil-de-usuario-no-un-system-prompt-condicional) y con el bloque hermano**: 🟩 GDA-Turnos usa `gda-turnos-ciudadano` / `gda-turnos-funcionario` — **perfil, no jurisdicción**.

### Consecuencias negativas

- 🟨 **Hay que corregir tres documentos, no uno.** El sufijo muerto `-{municipio}` (y su gemelo `boleteria-web-{municipio}`) está en:
  - `01-SAD.md` §6.6 y §10.3 (`:718-721`, `:1331`);
  - [`06-Administrator-Guide.md`](06-Administrator-Guide.md) §1.3, §3.2 y §8.3;
  - [`07-Plan-Sprints-Capacitacion.md`](07-Plan-Sprints-Capacitacion.md) `T-0.1` y `T-7.2`.

  🟩 Es trabajo declarado en `T-0.5`, cuyo alcance **debe cubrir los tres archivos**: 🟨 si `T-0.5` sólo enumera el SAD, el nombre muerto sobrevive en la guía del administrador — que es justamente el documento con el que se da de alta el tenant — y en el plan que ordena darlo de alta. **El alcance de esta supersesión es documental completo, no parcial.**
- 🟨 **Sin personalización por municipio.** Si Rosario quiere que el asistente mencione su Mesa de Ayuda propia, no hay dónde ponerlo salvo la KB común. 🟨 Mitigación: 🟩 `CONFIG_codMunicipio` es leíble por la API (`ParametrosService.cs:11-65`) ⇒ el dato local puede **inyectarse como contexto de tool**, no como tenant.
- 🟨 **Si mañana aparece un multi-municipio real, hay migración**: renombrar tenants implica tocar `lut_Tenants`, la KB y la config del widget. Costo aceptado a cambio de no pagar hoy la complejidad de un caso hipotético.
- 🟩 **`boleteria-web-{municipio}` queda sin definir.** El portal público está fuera del MVP ([ADR-011](#12-adr-011--alcance-del-mvp-diagnosticar-la-cadena-eventofunciónfuncionubicaciontarifa)); cuando entre, **este ADR aplica**: sería `boleteria-web-comprador`.

### Estado

🟨 **Propuesto.** ⚖️ **Supersede** todo uso del sufijo `-{municipio}`: `01-SAD.md` §6.6 y §10.3, `06-Administrator-Guide.md` §1.3/§3.2/§8.3 y `07-Plan-Sprints-Capacitacion.md` `T-0.1`/`T-7.2`. Depende de [ADR-009](#10-adr-009--dos-tenants-por-perfil-de-usuario-no-un-system-prompt-condicional). Relacionado con [ADR-003](#4-adr-003--propagación-de-identidad-token-exchange-de-la-cookie-del-backoffice) (el alcance real). **Cláusula de supersesión**: si se verifica una instalación que sirva a varios municipios con KB distintas, se reabre.

### Evidencia

| Afirmación | Fuente |
|---|---|
| BoleteriaCore no tiene multi-tenant; lo más cercano es `GP_IdMunicipio` | `SysVentaEntradasEventosModel.cs:23` · verdad de referencia §"Multi-tenant" |
| Que `GP_IdMunicipio` sea aislamiento es **inferencia**, no hecho | verdad de referencia §"No verificado" (🟨) |
| `lut_Parametros` es clave-valor global, sin scope ni tenant | `LutParametrosModel.cs:11-15` |
| `CONFIG_codMunicipio` se lee vía `ParametrosService` y se cachea en `IConfiguration` | `Services/ParametrosService.cs:11-65` |
| `lut_Tenants` define prompt/proveedor/modelo, **no** filtra datos | `IAConnect/scripts/01_create_database.sql:31-53` |
| 🟩 El SAD declara el sufijo `-{municipio}` (contradicho por este ADR) | [`01-SAD.md:718-721`](01-SAD.md) §6.6 · `:1331` |
| El bloque hermano nombra tenants por perfil, no por jurisdicción | [`../GDA-Turnos/04-ADR.md`](../GDA-Turnos/04-ADR.md) ADR-002 |

---

## 12. ADR-011 — Alcance del MVP: diagnosticar la cadena Evento→Función→FuncionUbicacion→Tarifa

### Contexto

🟩 La definición del caso es acotada y textual: *«guía para usuarios inexpertos en altas de eventos, funciones, tarifas… indicar ante una pregunta por qué el evento no se publicó qué configuración le faltó y dónde ir. Incluso generar un enlace puntual a la página donde configurar ese parámetro que faltó.»* 🟩 Con la aclaración: *«en especial es que eventos se relaciona con Funciones/Tarifas/parámetros»* — **la relación es el eje**.

🟩 Y la cadena tiene cuatro saltos con el precio en la tabla puente (**F1**). 🟨 Ahí está el caso entero: el usuario inexperto debe recorrer `Evento → Función → FuncionUbicacion → Tarifa` para entender por qué su evento no sale, y 🟩 la ia-db lo confirma — *«FuncionUbicacion es la tabla más importante del modelo»*.

🟨 El riesgo del MVP no es quedarse corto: es que «asistente de eventos» se expanda hasta ser «asistente de boletería», y con 🟩 cero function-calling en el gateway, cero tests en el anfitrión y sprocs opacos, un alcance amplio garantiza no entregar nada.

### Decisión

**El MVP es un asistente de sólo lectura, en `BoleteriaCore.Backoffice`, que responde una pregunta central — «¿por qué no se publicó este evento?» — recorriendo la cadena `Evento→Función→FuncionUbicacion→Tarifa`, devolviendo una causa de un enum cerrado y un deep-link a la pantalla exacta. Todo lo demás queda afuera de forma explícita y enumerada.**

```text
DENTRO del MVP
├── Host: BoleteriaCore.Backoffice ..................... 🟩 sólo este
├── Tenant: boleteria-backoffice-organizador .......... 🟩 ADR-010
├── Tools: T1…T6 ...................................... ⚖️ ADR-016
├── Diagnóstico: la cadena de 4 saltos ................ 🟩 F1
├── Causa: enum CausaNoPublicado ...................... ⚖️ ADR-017
├── Deep-link: DeepLinkBuilder ........................ 🟩 ADR-002
├── KB: conceptos + regla + procedimientos + pantallas  🟩 ADR-006
└── Hand-off: Mesa de Ayuda ........................... 🟩 MainLayout.razor:54

FUERA del MVP (enumerado, con motivo)
├── BoleteriaCore.Web (portal de compra) .............. 🟨 otra audiencia, otro caso
├── Escritura de cualquier tipo ....................... 🟨 ADR-007
├── Ventas / recaudación (resumen_ventas_evento) ...... 🟨 no es el caso; datos sensibles
├── Vigencia (verificar_vigencia_evento) .............. 🟩 BLOQUEADA: sprocs opacos (ADR-012)
├── Descuentos y combos ............................... 🟩 no participan de la publicación
├── Mapas / butacas / coordenadas ..................... 🟩 ParametrosMapasCoordenadas sin @page
├── Funciones ilimitadas .............................. 🟨 flujo no analizado; reglas propias
├── Cajeros / puntos de venta / usuarios .............. 🟨 otros módulos de "Parámetros"
└── Tenant admin (F2) ................................. 🟨 ADR-009, fase 2
```

🟨 **Tres exclusiones merecen justificación, porque un lector razonable las esperaría adentro:**

| Exclusión | Por qué duele | Por qué igual queda afuera |
|---|---|---|
| **Descuentos y combos** | Son "precio", y el caso es de precios. | 🟩 Son otro subsistema (`sys_Descuentos*`, `sys_DescuentoFuncionUbicacion`) y **no participan de la publicación**: la regla F3 evalúa `t.Precio > 0` sobre la tabla puente, nada más. Incluirlos agregaría superficie sin mover CE-1. |
| **Funciones ilimitadas** | Es un flujo de alta real que un organizador usa. | 🟨 `ParametrosEventosAltaFuncionesIlimitadas` / `FechaIlimitadaModel` **no fueron analizados**: 🟨 pueden tener reglas de publicación propias. Diagnosticar lo que no entendemos es inventar. ⇒ `Desconocida` + hand-off, y **verificación en sprint 0**. |
| **`lut_Parametros`** (la tabla) | El usuario pidió "parámetros" textualmente. | 🟩 **Ningún parámetro se valida como obligatorio antes de publicar** y la tabla no tiene FK a evento: está fuera del grafo. 🟨 Y hay una **ambigüedad de nombres verificada**: «Parámetros» en el Backoffice es el **módulo** de administración (`Components/Pages/Parametros/*`), no la tabla. 🟨 El *«parámetro que faltó»* del pedido **es un campo de configuración del evento** (un precio, una función activa), no una fila de `lut_Parametros`. **El MVP resuelve lo que el usuario quiso decir, no lo que la tabla se llama.** |

### Alternativas consideradas

| Alternativa | Atractivo | Por qué se descarta |
|---|---|---|
| **Asistente completo de boletería** (compra, ventas, accesos) | Un solo proyecto; máximo valor percibido; una sola aprobación. | 🟨 Con 🟩 cero function-calling, cero tests y sprocs opacos, es garantía de no entregar. 🟦 El primer caso de una plataforma de IA debe ser **angosto y demostrable**: su función es validar la plataforma, no agotar el dominio. ❌ |
| **Incluir el portal público** (`BoleteriaCore.Web`) | Mucho más volumen de usuarios; visibilidad. | 🟨 Otra audiencia (comprador), otra KB, otro tenant, otro riesgo (usuarios anónimos ⇒ superficie de abuso y costo por token no acotado). 🟨 Y **no es el caso pedido**: el pedido es de gestión, no de compra. 🟩 El SAD ya lo tenía en prioridad 2 (`01-SAD.md:719`). |
| **Sólo la KB, sin tools** («fase 0» informativa) | Entregable en semanas. Sin depender de [ADR-004](#5-adr-004--function-calling-genérico-en-iaconnect-no-un-hack-de-boletería). | 🟨 Entrega el modal que ya existe, en prosa. 🟩 No responde la pregunta del caso. 🟨 **Sería medido como fracaso del asistente** cuando en realidad es fracaso del alcance. ❌ |
| **Incluir ventas/recaudación** | 🟩 Estaba en el catálogo del HLD (`resumen_ventas_evento`, F2). | 🟨 Datos sensibles con una regla de alcance 🟨 **inferida** ([ADR-003](#4-adr-003--propagación-de-identidad-token-exchange-de-la-cookie-del-backoffice)). Exponer recaudación sobre un `alcance()` que nadie confirmó es un riesgo que el MVP no necesita correr. 🟩 Ya estaba marcada «⚠ diferida» (`02-HLD.md:1577`). |

### Consecuencias positivas

- 🟨 **El caso es demostrable en una sola pantalla**: modal de error → chat → causa → link → pantalla correcta.
- 🟨 **El alcance angosto hace medible CE-1.** Una sola pregunta central, un enum cerrado, muestra auditable a mano.
- 🟨 **Las exclusiones están enumeradas**, no implícitas: cuando alguien pida ventas, la respuesta es un ítem de lista con motivo, no una discusión.
- 🟨 **La cadena de 4 saltos es exactamente lo que el usuario pidió** — 🟩 *«en especial es que eventos se relaciona con Funciones/Tarifas/parámetros»*.

### Consecuencias negativas

- 🟨 **Va a parecer poco.** «¿Todo esto para responder una pregunta?» es la crítica previsible. La respuesta: 🟩 esa pregunta requiere function-calling que el gateway no tiene, una regla que nadie extrajo y un deep-link que nadie puede construir hoy.
- 🟩 **`Desconocida` va a dispararse más de lo previsto** por las exclusiones (funciones ilimitadas, sprocs). 🟨 El HLD ya lo anticipa. Es honesto, pero se ve como "el asistente no sabe".
- 🟨 **El portal público queda sin asistente**, que es donde está el volumen de usuarios.
- 🟨 **El MVP no arregla ninguna de las deudas que encontró** (`AccionPausar`, `Es_Referencia` sin mapear, `Id_Lugar` duplicado). Las **documenta**. 🟨 Para un stakeholder, encontrar bugs y no arreglarlos puede leerse como no entregar valor.

### Estado

🟨 **Propuesto.** Es la raíz del grafo: habilita ADR-004, 006, 007, 008, 009, 016. Revisable por [ADR-015](#16-adr-015--medición-del-éxito-y-criterio-de-continuidad-go--no-go).

### Evidencia

| Afirmación | Fuente |
|---|---|
| Definición textual del caso y aclaración sobre la relación | Pedido del usuario · [`01-SAD.md`](01-SAD.md) §2.1 |
| Cadena de 4 saltos con precio en la tabla puente | `SysTarifasModel.cs:11-33` · `SysTarifasUFuncionUbicacionModel.cs:17-19` |
| «FuncionUbicacion es la tabla más importante del modelo» | [`ia-db/indexes/02_Modelo-Dominio.md`](../../../../../BD/Boleteria.Core.Documentacion/ia-db/indexes/02_Modelo-Dominio.md) |
| Descuentos/combos no participan de la publicación | verdad de referencia §"Tarifas" · `sys_Descuentos*` |
| Ningún parámetro de `lut_Parametros` se valida antes de publicar | verdad de referencia §"lut_Parametros" (🟩) |
| «Parámetros» en el BO es el módulo de administración, no la tabla | `Components/Pages/Parametros/*` (🟨 ambigüedad verificada) |
| Funciones ilimitadas: flujo no analizado | verdad de referencia §"No verificado" (🟨) |
| `resumen_ventas_evento` ya estaba diferida | [`02-HLD.md:1577`](02-HLD.md) §12.1 |
| Portal público en prioridad 2 | [`01-SAD.md:719`](01-SAD.md) §6.6 |

---

## 13. ADR-012 — Stored procedures no verificables: se bloquea la capacidad, no se adivina

### Contexto

🟩 **Los cuerpos de los sprocs no están en el repositorio.** `DataManager/Migraciones/` contiene **dos** archivos: `issue-505.sql` (ALTERs) e `issue-506.sql` (un SP). 🟩 Sin verificar: `..._GetBy_Vigentes`, `..._GetBy_VigentesPV`, `..._GetBy_Id_EsFechaVigente`, `..._GetBy_Id_Evento_Vigentes`, `..._UpdateBy_Pausado`, `..._UpdateBy_AltaEventoCore`. Fuerza **F6**.

🟩 **Y no hay DDL**: no existe script de esquema; las FKs del ER están **inferidas** de campos `Id_*` y de los JOINs del único sproc disponible. 🟨 Cardinalidades exactas y existencia física de `FOREIGN KEY`: no verificadas.

🟨 El problema para el caso es directo: **cualquier regla de publicación embebida en SQL es invisible**. Si `sp_..._UpdateBy_Pausado` valida algo internamente, [ADR-005](#6-adr-005--la-regla-de-publicación-se-reimplementa-en-la-tool-con-test-de-equivalencia) lo desconoce y el diagnóstico puede decir «publicable» mientras el botón falla.

🟩 El HLD ya aplicó este criterio a un caso concreto: `verificar_vigencia_evento` está **bloqueada por evidencia, no por esfuerzo** (`02-HLD.md:1580-1584`). 🟨 Este ADR **generaliza** esa decisión puntual a política.

### Decisión

**Ninguna capacidad del asistente puede depender de un sproc cuyo cuerpo no esté en el repositorio y no haya sido leído. Ante un sproc opaco hay exactamente tres salidas permitidas, en este orden: (1) traer el cuerpo al repo, leerlo y desbloquear; (2) reimplementar la lógica en la API con test de equivalencia si la regla es verificable por otros medios; (3) **bloquear la capacidad y documentarlo**. Está prohibida la cuarta: inferir el comportamiento y programar contra la inferencia.**

```mermaid
flowchart TD
    N["Capacidad nueva<br/>que necesita un sproc"] --> Q{"¿El cuerpo está<br/>en el repo?"}
    Q -->|"sí"| OK["✅ Se implementa<br/>citando ruta:línea"]
    Q -->|"no"| Q2{"¿Se puede traer<br/>al repo y leerlo?"}
    Q2 -->|"sí"| T["Traerlo · versionarlo<br/>· leerlo → OK"]
    Q2 -->|"no"| Q3{"¿La regla es verificable<br/>por otra vía?"}
    Q3 -->|"sí"| R["Reimplementar + test<br/>de equivalencia (ADR-005)"]
    Q3 -->|"no"| B["🚫 BLOQUEAR<br/>+ documentar<br/>+ hand-off"]
    Q3 -.->|"❌ PROHIBIDO"| I["Inferir el<br/>comportamiento"]

    classDef bad fill:#fce8e6,stroke:#d93025,stroke-width:3px,color:#000
    classDef ok fill:#e6f4ea,stroke:#137333,color:#000
    classDef warn fill:#fef7e0,stroke:#f9ab00,color:#000
    class I bad
    class OK,T,R ok
    class B warn
```

| Capacidad | Sproc | Estado | Salida aplicada |
|---|---|---|---|
| Diagnóstico de publicación (T1) | lectura vía DataManagers | ✅ | (2) Reimplementar + test — [ADR-005](#6-adr-005--la-regla-de-publicación-se-reimplementa-en-la-tool-con-test-de-equivalencia) |
| `verificar_vigencia_evento` | `..._GetBy_Vigentes`, `..._GetBy_Id_EsFechaVigente` | 🚫 | (3) **Bloqueada** 🟩 `02-HLD.md:1580-1584` |
| Publicar/pausar | `..._UpdateBy_Pausado` | 🚫 | (3) Fuera de alcance — [ADR-007](#8-adr-007--el-asistente-no-ejecuta-acciones-tools-de-sólo-lectura-en-la-v1) |
| Alta de evento | `..._UpdateBy_AltaEventoCore` | 🚫 | (3) Fuera de alcance — [ADR-011](#12-adr-011--alcance-del-mvp-diagnosticar-la-cadena-eventofunciónfuncionubicaciontarifa) |

🟨 **Por qué «bloquear» y no «devolver el booleano igual»**: 🟩 el HLD ya dio la razón exacta y vale como principio general — *«se podría invocar y devolver el booleano, pero el asistente no podría explicar el porqué, y un booleano sin porqué es, otra vez, el modal»*. El caso **existe** para explicar el porqué. Una capacidad que no puede explicarse no cumple el caso aunque devuelva el dato correcto.

### Alternativas consideradas

| Alternativa | Atractivo | Por qué se descarta |
|---|---|---|
| **Inferir el comportamiento del sproc por su nombre y sus usos** | Desbloquea todo. Los nombres son descriptivos (`GetBy_Vigentes`). Es lo que haría cualquiera con prisa. | 🟨 **Es alucinación con pasos extra.** 🟩 `DataEntityCore` bindea **posicionalmente** tras `DeriveParameters` (`DataEntityCore.cs:18-27,43-46`) ⇒ ni siquiera la firma es estable a la lectura del C#. 🟨 Y programar contra una inferencia, en un asistente cuyo valor **es** no inventar, sería contradecir el producto en su propia implementación. ❌ |
| **Extraer los sprocs de la base productiva y versionarlos** | 🟦 **Es lo correcto.** Resuelve el problema de raíz, para este caso y para todos. | 🟨 No se descarta: es la salida **(1)**, y este ADR la prioriza. 🟨 Se descarta como **precondición** del MVP porque depende de acceso a producción, de un dueño de base y de una decisión de gobierno que este proyecto no controla. 🟨 **Se registra como recomendación fuerte al equipo de BoleteriaCore**, con impacto que excede a la IA. |
| **Leer los sprocs sólo en runtime** (`sys.sql_modules`) para el diagnóstico | Sin proceso de gobierno. La API los leería sola. | 🟨 Ingenioso y peligroso: la lógica del asistente dependería de texto SQL parseado en caliente. 🟨 No testeable en CI, no reproducible, y rompería con cualquier cambio de formato. ❌ |
| **Asumir que los sprocs no tienen lógica de negocio** | 🟩 Cómodo: el patrón observado es CRUD por convención. | 🟨 «Probablemente sea CRUD» no es evidencia. 🟩 Y hay señal en contra: 🟩 `AccionPausar` llama `UpdateByPausado` sin validar del lado cliente — si algo protege ese camino, **está en el sproc**. Asumir que no hay nada es asumir que el sistema está roto; puede ser cierto, pero **hay que verificarlo, no suponerlo**. ❌ |

### Consecuencias positivas

- 🟨 **El asistente nunca afirma lo que no puede verificar.** Es CE-8 aplicado a la arquitectura, no sólo a las respuestas.
- 🟨 **Las capacidades bloqueadas son visibles y accionables**: cada una tiene el sproc que la desbloquearía. Es un backlog, no un misterio.
- 🟨 **Crea presión sana sobre BoleteriaCore.** 🟩 Que un proyecto de IA no pueda avanzar porque los sprocs no están versionados es un argumento concreto para versionarlos — con beneficio que excede a la IA.
- 🟦 **La política es simple de auditar en code review**: ¿esta tool depende de un sproc? ¿está su cuerpo en el repo? Dos preguntas.

### Consecuencias negativas

- 🟩 **Se pierde vigencia**, que es una pregunta legítima y frecuente (*«¿está vigente mi evento?»*). El asistente deriva.
- 🟨 **La política es conservadora y puede bloquear de más.** Algunos sprocs seguramente son CRUD inocente; los tratamos como opacos igual. **Costo aceptado**: el falso negativo (bloquear algo inocuo) cuesta una capacidad; el falso positivo (habilitar algo con lógica oculta) cuesta la confianza.
- 🟨 **`Desconocida` crece.** Junto con [ADR-011](#12-adr-011--alcance-del-mvp-diagnosticar-la-cadena-eventofunciónfuncionubicaciontarifa), es la segunda fuente de hand-offs.
- 🟩 **Ni siquiera [ADR-005](#6-adr-005--la-regla-de-publicación-se-reimplementa-en-la-tool-con-test-de-equivalencia) queda a salvo.** El diagnóstico lee vía DataManagers que invocan sprocs opacos: **si un `GetBy_*` filtra filas por una regla que no vemos, el traversal ve un subconjunto y diagnostica sobre datos incompletos.** 🟨 Este ADR **acota el riesgo; no lo elimina**, y es honesto decirlo: es el límite duro del bloque.

### Estado

🟨 **Propuesto.** Limita a [ADR-005](#6-adr-005--la-regla-de-publicación-se-reimplementa-en-la-tool-con-test-de-equivalencia) y [ADR-011](#12-adr-011--alcance-del-mvp-diagnosticar-la-cadena-eventofunciónfuncionubicaciontarifa). 🟩 Formaliza RA-5 del SAD y generaliza el bloqueo puntual de `02-HLD.md:1580-1584`.

### Evidencia

| Afirmación | Fuente |
|---|---|
| El repo sólo tiene 2 archivos SQL; cuerpos de sprocs ausentes | `DataManager/Migraciones/issue-505.sql`, `issue-506.sql` |
| Lista de sprocs sin verificar | verdad de referencia §"No verificado" |
| Sin DDL: FKs y cardinalidades inferidas de campos `Id_*` y JOINs | verdad de referencia §"No verificado" |
| `verificar_vigencia_evento` bloqueada por evidencia, no por esfuerzo | [`02-HLD.md:1580-1584`](02-HLD.md) §12.1 |
| «Un booleano sin porqué es, otra vez, el modal» | [`02-HLD.md:1584`](02-HLD.md) |
| `DataEntityCore` bindea posicionalmente tras `DeriveParameters` | `Notions.Core.Utils.DataManager/DataEntityCore.cs:18-27,43-46` |
| `AccionPausar` llama `UpdateByPausado` sin validación client-side | `ParametrosEventos.razor.cs:441-461` · `SysVentaEntradasEventosDataManager.cs:32-42` |
| Los DAOs invocan sprocs por convención (`GetByAsync("Vigentes")`) | verdad de referencia §"Patrón de datos" |

---

## 14. ADR-013 — Curaduría y propiedad de la KB: dueño funcional + pipeline idempotente

### Contexto

🟨 La KB no es documentación: es **código de producción en prosa**. Un párrafo mal escrito no queda feo — hace que el asistente le diga algo falso a un usuario inexperto que no tiene cómo detectarlo.

🟩 Y el motor la castiga: TF-IDF top-5 **sin threshold** (`RAGEngine.cs:34-127`). 🟨 Consecuencias directas sobre **cómo hay que escribirla**: (a) el vocabulario importa más que la prosa — si el usuario dice *«obra»* y la KB dice *«evento»*, la similitud es baja; (b) documentos largos diluyen su propio TF-IDF; (c) como siempre entran 5 fragmentos, **cada documento irrelevante compite por un lugar en el prompt en todas las consultas**. 🟨 En un RAG léxico, **agregar un documento puede empeorar respuestas que no tienen nada que ver con él.**

🟨 Y hay un vocabulario doble que la KB debe reconciliar: 🟩 «Publicado» no existe en la base (**F2**), 🟩 «Parámetros» es un módulo y también una tabla, 🟩 «Función» es lo que el usuario llama *fecha* o *presentación*.

🟩 `PromptBuilder` interpola los fragmentos **sin escapado** (`PromptBuilder.cs:10-55`) ⇒ **la KB es superficie de prompt injection**: quien pueda escribir en ella puede reescribir el comportamiento del asistente.

### Decisión

**La KB tiene un dueño funcional nombrado — el responsable funcional de Boletería, no el equipo de IA — que aprueba cada cambio de contenido. La KB se versiona en Git como archivos Markdown en el repositorio de documentación, fuente única, y se publica a IAConnect mediante un pipeline idempotente (upsert por `slug` + hash) disparado por merge a `main`. Nadie edita documentos directamente en IAConnect en producción: la base de conocimiento es un artefacto de build, no un formulario. Se escribe léxico-first, con sinónimos del usuario incrustados en el texto.**

```mermaid
flowchart LR
    A["Autor funcional<br/>escribe .md"] --> PR["Pull Request"]
    PR --> RV{"Review"}
    RV -->|"dueño funcional:<br/>¿es verdad?"| RV2{"IA:<br/>¿es léxico-first?"}
    RV2 --> M["merge a main"]
    M --> P["Pipeline<br/>upsert por slug+hash"]
    P --> T1["KB tenant<br/>organizador"]
    P --> T2["KB tenant<br/>admin"]
    P --> EV["⚠ Suite de evaluación<br/>10 diálogos del HLD"]
    EV -->|"falla"| RB["rollback:<br/>revert + re-publish"]
    X["✋ edición manual<br/>en producción"] -.->|"❌ prohibida"| T1

    classDef bad fill:#fce8e6,stroke:#d93025,color:#000
    classDef ok fill:#e6f4ea,stroke:#137333,color:#000
    class X,RB bad
    class M,P,EV ok
```

🟨 **Reglas de redacción léxico-first** (derivadas de 🟩 el motor, no de gusto):

| # | Regla | Motivo |
|---|---|---|
| K1 | Un documento = una pregunta del usuario | 🟩 TF-IDF: documentos chicos, señal alta |
| K2 | El título es la pregunta, con las palabras del usuario | 🟨 El título pesa en la similitud léxica |
| K3 | **Sinónimos explícitos en el cuerpo** (*«función (también: fecha, presentación, show)»*) | 🟩 No hay embeddings: el sinónimo debe estar **literal** |
| K4 | Ambos vocabularios (UI y base): *«Publicado (en la base: `Pausado = false`)»* | 🟩 F2 |
| K5 | **Cero datos de eventos concretos** | 🟩 [ADR-006](#7-adr-006--arquitectura-de-conocimiento-híbrida-rag-para-lo-estable-tools-para-lo-volátil) |
| K6 | **Cero valores de lookup** | 🟩 `06-Administrator-Guide.md:823` |
| K7 | Cada documento nuevo justifica **por qué no empeora el top-5** | 🟨 Sin threshold, todo documento compite siempre |
| K8 | Sin instrucciones imperativas al modelo en el cuerpo | 🟩 `PromptBuilder` no escapa: es el vector de injection |

### Alternativas consideradas

| Alternativa | Atractivo | Por qué se descarta |
|---|---|---|
| **KB editable en la UI de IAConnect por el área funcional** | 🟩 IAConnect **ya tiene** ese ABM. Cero pipeline. Autonomía total del negocio; correcciones en minutos. | 🟨 **La KB es código y esto es editar producción sin PR.** Sin diff, sin review, sin rollback, sin trazabilidad de quién rompió qué. 🟩 Y con `PromptBuilder` sin escapado, un párrafo mal pegado es **prompt injection persistente**. 🟨 Se conserva como **procedimiento de emergencia documentado** ([`05-Operations-Guide.md`](05-Operations-Guide.md)): permitido para *deshabilitar* un documento dañino, nunca para agregar contenido, y con reconciliación obligatoria a Git en 24 h. |
| **La KB la escribe y aprueba el equipo de IA** | Rápido; el equipo conoce el motor y las reglas K1-K8. | 🟨 **El equipo de IA no sabe si el contenido es verdad.** 🟨 Y ya se equivocó de forma documentada: 🟩 las tres incoherencias A/B/C de este bloque las produjo este equipo escribiendo sobre su propio dominio. Si nos equivocamos en los nombres de nuestras propias tools, no somos la autoridad sobre las reglas de negocio de boletería. La verificación factual necesita al dueño funcional. |
| **Generar la KB automáticamente desde el código** | Nunca se desactualiza. Trazable por construcción. | 🟨 El código dice *qué* valida (🟩 `t.Precio > 0`), no *por qué* ni *qué hacer*. La KB existe para explicarle a un inexperto lo que el modal ya le dijo y no entendió: **generarla del código reproduce el modal**. 🟨 Sí se adopta **parcialmente**: el mapa de pantallas se genera de 🟩 `routes-map.md` y un test lo contrasta con los `@page` reales. |
| **Sin KB: todo en el system prompt** | Simple; sin RAG; sin curaduría; sin pipeline. | 🟨 No escala y se paga en cada token de cada request. 🟩 Y `lut_Tenants.System_Prompt` es una columna: el vocabulario, la regla, los procedimientos y el mapa de pantallas no entran. ❌ |

### Consecuencias positivas

- 🟨 **La verdad factual la aprueba quien la conoce; la forma la revisa quien conoce el motor.** Dos revisores, dos preguntas distintas, ninguna redundante.
- 🟦 **Rollback real.** `git revert` + re-publish. Con edición manual en producción, el rollback es "acordate de qué decía antes".
- 🟨 **Idempotencia** ⇒ el pipeline se puede correr N veces sin duplicar. 🟩 Importa: la KB va a **dos** tenants ([ADR-009](#10-adr-009--dos-tenants-por-perfil-de-usuario-no-un-system-prompt-condicional)) desde una fuente única.
- 🟨 **K7 institucionaliza el hecho más contraintuitivo del motor**: en un RAG sin threshold, **menos es más**. Sin esa regla, la KB crece por acumulación y la precisión baja sin que nadie entienda por qué.
- 🟨 **La suite de evaluación convierte "mejoré la KB" en una afirmación medible.** 🟩 Los 10 diálogos del HLD son la línea de base.

### Consecuencias negativas

- 🟨 **Fricción real.** Un typo requiere PR, review y pipeline. El área funcional va a pedir el botón de editar. La válvula de emergencia existe y **hay que auditarla**, o se vuelve el camino normal.
- 🟨 **Depende de que exista un dueño funcional nombrado y con tiempo.** 🟨 **Si no lo hay, este ADR no se cumple** y la KB termina siendo del equipo de IA por default — que es exactamente lo que la alternativa descartada proponía, pero sin haberlo decidido. **Es una precondición organizacional, no técnica, y es el mayor riesgo del ADR.**
- 🟨 **K3 (sinónimos literales) es mantenimiento manual perpetuo.** Cada palabra nueva que use un usuario es un cambio en la KB. 🟨 Es el peaje de no tener embeddings, y va a doler.
- 🟨 **La suite de evaluación hay que construirla y mantenerla.** Sin ella el pipeline publica cualquier cosa con prolijidad.

### Estado

🟨 **Propuesto.** Depende de [ADR-006](#7-adr-006--arquitectura-de-conocimiento-híbrida-rag-para-lo-estable-tools-para-lo-volátil) y [ADR-009](#10-adr-009--dos-tenants-por-perfil-de-usuario-no-un-system-prompt-condicional). El procedimiento operativo lo detalla [`05-Operations-Guide.md`](05-Operations-Guide.md); el instructivo de autoría, [`06-Administrator-Guide.md`](06-Administrator-Guide.md).

### Evidencia

| Afirmación | Fuente |
|---|---|
| RAG léxico TF-IDF top-5 sin threshold | `IAConnect.Application/Services/RAGEngine.cs:34-127` |
| `PromptBuilder` interpola la KB sin escapado (vector de injection) | `IAConnect.Application/Services/PromptBuilder.cs:10-55` |
| «Publicado» no existe en la base; es `!Pausado` | `ParametrosEventosEdit.razor.cs:174` |
| Instrucción de no listar lookups en la KB | [`06-Administrator-Guide.md:823`](06-Administrator-Guide.md) |
| Los 10 diálogos como línea de base de evaluación | [`02-HLD.md:1778`](02-HLD.md) |
| Mapa de pantallas verificable contra rutas reales | [`routes-map.md`](../../../../../BD/Boleteria.Core.Documentacion/boleteria-core-docs/docs/pieces/boleteria-core-backoffice/routes-map.md) |
| El system prompt es una columna de `lut_Tenants` | `IAConnect/scripts/01_create_database.sql:31-53` |

---

## 15. ADR-014 — Fallback ante proveedor LLM caído: degradación determinística, no failover

### Contexto

🟩 El proveedor y el modelo son columnas del tenant (`scripts/01_create_database.sql:31-53`) ⇒ IAConnect es multi-proveedor **por configuración**.

🟨 La observación que ordena este ADR: **el asistente tiene dos mitades con disponibilidad independiente.** La mitad que **sabe** es la tool — 🟩 determinista, sin LLM, contra la base de BoleteriaCore. La mitad que **habla** es el LLM. 🟨 Si se cae el LLM, el diagnóstico **sigue siendo calculable**: `CausaNoPublicado.TarifasSinPrecio` + deep-link no requieren un modelo. Lo único que se pierde es la redacción.

🟨 Y una asimetría útil: 🟩 el veredicto ya es un enum cerrado con deep-link asociado ([ADR-017](#18-adr-017--️-nomenclatura-canónica-del-enum-causanopublicado-resuelve-incoherencia-b)) ⇒ **hay un texto correcto por causa, escribible de antemano**. El LLM personaliza; no descubre.

### Decisión

**Ante proveedor LLM caído no se hace failover automático a otro proveedor: se degrada a modo determinístico. El widget invoca la tool directamente y renderiza una plantilla fija por valor de `CausaNoPublicado`, con su deep-link, indicando de forma visible que el asistente conversacional no está disponible. Las preguntas que requieren RAG o redacción libre se derivan a Mesa de Ayuda. El failover manual entre proveedores existe como decisión de operación, con cambio de configuración de tenant y validación previa, nunca automático.**

```mermaid
stateDiagram-v2
    [*] --> Normal
    Normal --> Degradado: LLM 5xx / timeout / circuito abierto
    Degradado --> Normal: health-check OK · 3 éxitos seguidos

    state Normal {
        [*] --> Chat
        Chat --> Tool
        Tool --> Redaccion: LLM redacta
    }
    state Degradado {
        [*] --> ToolD: widget → tool directo
        ToolD --> Plantilla: texto fijo por CausaNoPublicado
        Plantilla --> Link: deep-link intacto
        ToolD --> HandOff: si la pregunta era de RAG
    }
    note right of Degradado
        🟨 El diagnóstico NO depende del LLM.
        Se pierde el tono, no el veredicto.
    end note
```

| Falla | Detección | Respuesta | Marca |
|---|---|---|---|
| LLM 5xx / timeout | Circuit breaker en IAConnect | Modo degradado | 🟨 |
| LLM lento (> 10 s) | Timeout de request | Modo degradado | 🟨 |
| Rate limit del proveedor | 429 | Backoff; si persiste, degradado | 🟦 |
| Tool caída (AI.Api 5xx) | HTTP | **No hay degradación posible**: sin tool no hay caso ⇒ hand-off | 🟨 |
| Proveedor sin function-calling | Config de tenant | Bloqueo de despliegue, no runtime | 🟩 |

🟨 **La fila que importa es la cuarta**: si se cae la API adaptadora, no hay modo degradado — **la tool es el caso**. El LLM es prescindible; la tool no. Es lo contrario de lo que intuye cualquiera que mire un chatbot.

### Alternativas consideradas

| Alternativa | Atractivo | Por qué se descarta |
|---|---|---|
| **Failover automático a otro proveedor** | 🟦 Patrón clásico de alta disponibilidad. Continuidad transparente. IAConnect ya es multi-proveedor. | 🟨 **El proveedor no es un detalle intercambiable: es parte del comportamiento.** 🟩 El system prompt, la temperatura y el modelo son del tenant y están afinados **contra un proveedor**; el catálogo de tools se declara con **su** dialecto de function-calling ([ADR-004](#5-adr-004--function-calling-genérico-en-iaconnect-no-un-hack-de-boletería)). 🟨 Un failover silencioso cambia el comportamiento del asistente en el peor momento — durante un incidente, sin evaluación previa, con el prompt afinado para otro. 🟨 Peor: **puede degradar sin caerse**, que es indetectable para el usuario y para CE-1. ❌ como automático. |
| **Sin fallback: "el asistente no está disponible"** | Honesto y trivial. Cero código. | 🟨 **Desperdicia la mitad que funciona.** El diagnóstico no necesita LLM: 🟩 la tool es determinista. Decirle «no disponible» a un usuario cuya respuesta ya está calculada es una decisión de ingeniería perezosa. ❌ |
| **Cachear respuestas del LLM** | Continuidad para preguntas frecuentes. | 🟨 **Las respuestas del caso son por evento y por estado.** Una respuesta cacheada de *«¿por qué no se publicó el 42?»* es exactamente el dato rancio que 🟩 [ADR-006](#7-adr-006--arquitectura-de-conocimiento-híbrida-rag-para-lo-estable-tools-para-lo-volátil) prohíbe: el usuario cargó el precio y el caché insiste en que falta. ❌ |
| **Modelo local de respaldo** | Sin dependencia externa. | 🟨 Infraestructura, GPU y operación desproporcionadas para redactar 7 plantillas que ya podemos escribir a mano. ❌ |

### Consecuencias positivas

- 🟨 **El caso sobrevive a la caída del LLM.** Se pierde el tono; el veredicto y el deep-link siguen. 🟨 Es posiblemente la mejor consecuencia de haber elegido **enum + datos, nunca prosa** (🟩 `02-HLD.md:1586-1612`): esa decisión de diseño **paga acá**.
- 🟦 **Sin cambios de comportamiento silenciosos.** El proveedor cambia por decisión humana, con evaluación previa.
- 🟨 **La degradación es visible al usuario.** 🟦 Un asistente degradado que finge normalidad es peor que uno que avisa.
- 🟨 **Las plantillas por causa son testeables** y sirven de línea de base: si el LLM redacta peor que la plantilla, hay un problema de prompt.

### Consecuencias negativas

- 🟨 **Hay que escribir y mantener las plantillas** (una por valor del enum) — una tercera superficie de texto, además de KB y system prompt. Si el enum cambia, hay que actualizarlas.
- 🟨 **El modo degradado no entiende preguntas.** Sólo funciona si el widget puede inferir el `idEvento` del contexto de pantalla ([ADR-008](#9-adr-008--widget-como-componente-blazor-en-mainlayout-no-script-de-cdn)). Fuera de una pantalla de evento, degradado ≈ hand-off.
- 🟨 **El failover manual necesita runbook y gente entrenada** ([`05-Operations-Guide.md`](05-Operations-Guide.md)). En una caída de madrugada, "manual" significa "no pasa nada hasta la mañana".
- 🟩 **Un proveedor sin function-calling no tiene fallback posible**: sería un tenant que no puede ejecutar el caso. Por eso se valida en el **despliegue**, no en runtime.
- 🟨 **El circuit breaker hay que construirlo.** No verificado que IAConnect tenga uno.

### Estado

🟨 **Propuesto.** Depende de [ADR-004](#5-adr-004--function-calling-genérico-en-iaconnect-no-un-hack-de-boletería) y [ADR-017](#18-adr-017--️-nomenclatura-canónica-del-enum-causanopublicado-resuelve-incoherencia-b). Procedimientos en [`05-Operations-Guide.md`](05-Operations-Guide.md).

### Evidencia

| Afirmación | Fuente |
|---|---|
| Proveedor, modelo y temperatura son columnas del tenant | `IAConnect/scripts/01_create_database.sql:31-53` |
| Principio «enum + datos, nunca prosa» ⇒ el veredicto no depende del LLM | [`02-HLD.md:1586-1612`](02-HLD.md) §12.2 |
| El enum es cerrado y cada causa tiene deep-link asociado | [`02-HLD.md:1614-1624`](02-HLD.md) §12.3 |
| Hand-off a Mesa de Ayuda (ítem existente del sidebar) | `MainLayout.razor:54` · [`02-HLD.md:1197`](02-HLD.md) |
| IAConnect no tiene function-calling hoy (dialecto por proveedor) | grep exhaustivo · [`01-SAD.md`](01-SAD.md) §3.3 (RI-1) |

---

## 16. ADR-015 — Medición del éxito y criterio de continuidad (go / no-go)

### Contexto

🟨 El riesgo de un piloto de IA no es que fracase: es que **nadie pueda decidir si fracasó**. Sin umbral acordado de antemano, el resultado se interpreta según quién lo mire — y un asistente que responde fluido *parece* exitoso aunque acierte el 60%.

🟩 El HLD ya definió las métricas (CE-1…CE-8) y el criterio medible clave: 🟩 CE-1 = *«`CausaNoPublicado` de la tool == causa real (auditada a mano sobre muestra), ≥ 95%»* (`02-HLD.md:1679`). 🟨 Lo que falta no es la métrica: es **la consecuencia de no alcanzarla**.

🟨 Y hay una trampa de medición específica del caso: **el enum hace medible la tool, no el asistente.** Un `CausaNoPublicado` correcto redactado de forma que el usuario no entiende es un fracaso con CE-1 en verde. Por eso hace falta una métrica de resultado, no sólo de acierto.

### Decisión

**Se mide con dos métricas de decisión, no ocho. La primaria es la **tasa de resolución**: proporción de diagnósticos que terminan con el usuario navegando el deep-link y publicando el evento dentro de la hora, medida contra la línea de base previa al asistente. La secundaria y bloqueante es CE-1 ≥ 95%. Se evalúa a los 3 meses de uso real con umbrales fijados **antes** del despliegue, y la decisión tiene tres salidas predefinidas: escalar, iterar o abandonar. «Abandonar» es un resultado aceptable y explícitamente permitido.**

| Métrica | Definición | Umbral | Rol |
|---|---|---|---|
| **M1 · Resolución** | Diagnóstico → deep-link → publicado, < 1 h | ≥ **40%** 🟨 | **Decisión** |
| **M2 · Acierto (CE-1)** | 🟩 `CausaNoPublicado` == causa real, muestra auditada | ≥ **95%** 🟩 | **Bloqueante** |
| M3 · Deep-link válido (CE-2) | HTTP 200 + pantalla correcta | 100% | Higiene |
| M4 · Hand-off | % de `Desconocida` + fallback | ≤ 15% 🟨 | Diagnóstico del alcance |
| M5 · No inventar (CE-8) | Afirmaciones sin tool sobre evento concreto | 0 | **Veto** |

```mermaid
flowchart TD
    E["3 meses de uso real"] --> M5{"M5 = 0?<br/>(no inventa)"}
    M5 -->|"no"| STOP["🚫 VETO<br/>Se apaga hasta corregir"]
    M5 -->|"sí"| M2{"M2 ≥ 95%?"}
    M2 -->|"no"| IT2["ITERAR<br/>el enum está incompleto<br/>→ ADR-005, ADR-011"]
    M2 -->|"sí"| M1{"M1 ≥ 40%?"}
    M1 -->|"sí"| GO["✅ ESCALAR<br/>fase 2: tenant admin,<br/>portal, ¿escritura?"]
    M1 -->|"20–40%"| IT["ITERAR<br/>1 trimestre más,<br/>umbral fijo"]
    M1 -->|"< 20%"| NO["🛑 ABANDONAR<br/>y documentar por qué"]
    IT --> E

    classDef bad fill:#fce8e6,stroke:#d93025,stroke-width:2px,color:#000
    classDef ok fill:#e6f4ea,stroke:#137333,stroke-width:2px,color:#000
    classDef warn fill:#fef7e0,stroke:#f9ab00,color:#000
    class STOP,NO bad
    class GO ok
    class IT,IT2 warn
```

🟨 **M1 exige una línea de base que hoy no existe.** Hay que medir **antes** de desplegar: cuántos eventos quedan pausados con la regla incumplida y cuánto tardan en resolverse. 🟨 **Sin línea de base, M1 no es interpretable y este ADR no se puede aplicar.** Es trabajo del sprint 0 y **precondición del despliegue**, no del análisis posterior.

### Alternativas consideradas

| Alternativa | Atractivo | Por qué se descarta |
|---|---|---|
| **Satisfacción del usuario** (👍/👎 en el chat) | 🟦 Estándar; barato; feedback cualitativo directo. | 🟨 **Mide simpatía, no corrección.** Un asistente amable que se equivoca cosecha 👍 de usuarios inexpertos que **no pueden evaluar la respuesta** — es la definición de la audiencia. 🟨 Se conserva como señal cualitativa para priorizar la KB; **nunca como criterio de continuidad**. |
| **Volumen de uso** (consultas/día) | Fácil de instrumentar; se ve bien en un tablero. | 🟨 **Ambiguo por diseño**: mucho uso puede significar que ayuda, o que el Backoffice es tan confuso que todos preguntan. 🟨 Y sube con la novedad. Métrica de vanidad. ❌ |
| **Las 8 CE del HLD como criterio conjunto** | 🟩 Ya están definidas y son buenas. | 🟨 Ocho métricas no deciden nada: siempre habrá 5 en verde para defender el proyecto y 3 en rojo para atacarlo. 🟨 Este ADR **no las reemplaza** — las conserva como instrumentación y **elige dos para decidir**. |
| **Evaluación continua sin fecha de corte** | Flexible; evita cortar algo que estaba por mejorar. | 🟨 Es cómo los pilotos sobreviven sin entregar valor: siempre falta una iteración. 🟦 Una fecha y un umbral fijados **antes** son la única defensa contra el sesgo de costo hundido. ❌ |
| **A/B contra un grupo sin asistente** | 🟦 Rigor causal real. | 🟨 Población chica (usuarios del BO de un municipio) ⇒ sin poder estadístico. 🟨 Y hay un problema ético menor: negarle deliberadamente ayuda a un grupo de usuarios internos. Se prefiere pre/post con línea de base. |

### Consecuencias positivas

- 🟨 **La decisión está tomada antes de tener el resultado**, que es la única forma de tomarla honestamente.
- 🟨 **«Abandonar» es una salida legítima y escrita.** 🟨 Un caso que puede fracasar es un caso del que se puede aprender; uno que no puede fracasar es marketing.
- 🟨 **M5 es un veto, no un umbral.** 🟩 Con `PromptBuilder` sin escapado y RAG sin threshold, inventar es el modo de falla estructural del stack: no se negocia con un porcentaje.
- 🟨 **M4 diagnostica el alcance, no el asistente.** 🟩 Si `Desconocida` explota, la causa probable son las exclusiones de [ADR-011](#12-adr-011--alcance-del-mvp-diagnosticar-la-cadena-eventofunciónfuncionubicaciontarifa)/[ADR-012](#13-adr-012--stored-procedures-no-verificables-se-bloquea-la-capacidad-no-se-adivina) (funciones ilimitadas, sprocs opacos), no el modelo. La métrica apunta al ADR correcto.
- 🟨 **Escalar también está definido**: fase 2 = tenant admin + portal + reconsiderar [ADR-007](#8-adr-007--el-asistente-no-ejecuta-acciones-tools-de-sólo-lectura-en-la-v1) y la extracción del Service compartido de [ADR-005](#6-adr-005--la-regla-de-publicación-se-reimplementa-en-la-tool-con-test-de-equivalencia).

### Consecuencias negativas

- 🟨 **Los umbrales (40%, 95%, 15%) son juicio, no ciencia.** 🟨 No hay dato previo que los respalde — 🟩 CE-1 ≥ 95% viene del HLD, el resto se propone acá. **Deben acordarse con el responsable funcional antes del despliegue o no valen.**
- 🟨 **M1 requiere correlacionar chat con publicación efectiva** ⇒ telemetría que cruza IAConnect con BoleteriaCore. Trabajo real, y 🟩 con implicancias de privacidad (transcripts + identidad).
- 🟨 **Atribución imperfecta.** Si el usuario publica dentro de la hora, no sabemos si fue por el asistente. Aceptamos el ruido: la comparación es contra la línea de base, no evento por evento.
- 🟨 **Auditar CE-1 a mano no escala.** Es muestreo, con el sesgo del muestreo.
- 🟨 **3 meses puede ser poco** si la adopción es lenta. 🟨 Mitigación: la ventana arranca cuando se alcanza un mínimo de volumen, no en el día del deploy.

### Estado

🟨 **Propuesto.** Formaliza CE-1…CE-8 del HLD. Puede revertir [ADR-011](#12-adr-011--alcance-del-mvp-diagnosticar-la-cadena-eventofunciónfuncionubicaciontarifa) y habilitar la revisión de [ADR-007](#8-adr-007--el-asistente-no-ejecuta-acciones-tools-de-sólo-lectura-en-la-v1) y [ADR-005](#6-adr-005--la-regla-de-publicación-se-reimplementa-en-la-tool-con-test-de-equivalencia). **Los umbrales requieren acuerdo formal previo al despliegue.**

### Evidencia

| Afirmación | Fuente |
|---|---|
| CE-1: `CausaNoPublicado` == causa real, ≥ 95%, auditado sobre muestra (⚖️ el HLD lo llama `CausaCode`; corregido por ADR-017) | [`02-HLD.md:1679`](02-HLD.md) §13 |
| Catálogo CE-1…CE-8 | [`02-HLD.md`](02-HLD.md) §13 · [`01-SAD.md`](01-SAD.md) §2.5 |
| CE-2: deep-link válido, testeable en CI contra rutas reales | [`routes-map.md`](../../../../../BD/Boleteria.Core.Documentacion/boleteria-core-docs/docs/pieces/boleteria-core-backoffice/routes-map.md) · ADR-002 |
| `Desconocida` es valor de primera clase y va a dispararse | [`02-HLD.md:1624,1792-1797`](02-HLD.md) |
| `PromptBuilder` sin escapado; RAG sin threshold (⇒ M5 es veto) | `PromptBuilder.cs:10-55` · `RAGEngine.cs:34-127` |
| Riesgo H7: funciones ilimitadas ⇒ enum incompleto | [`02-HLD.md:1792`](02-HLD.md) |

---

## 17. ADR-016 — ⚖️ Catálogo canónico de tools: T1…T6 (resuelve incoherencia **A**)

> ⚖️ **ADR de desempate.** 🟩 Divergencia verificada entre tres documentos del propio bloque (`01-SAD.md:619-622` vs. `02-HLD.md:1572-1578` vs. `03-LLD.md:38-46`). **Este ADR fija el catálogo. Los demás documentos se corrigen.**

### Contexto

🟩 Tres documentos declaran tres catálogos distintos para las mismas capacidades:

| Documento | Catálogo declarado |
|---|---|
| 🟩 `01-SAD.md` §6.3 (`:619-622`) | `diagnosticar_evento` · `listar_mis_eventos` · `detalle_funcion` · `explicar_regla` |
| 🟩 `02-HLD.md` §12.1 (`:1572-1578`) | `buscar_evento` · `diagnosticar_publicacion` · `detalle_configuracion_evento` · `explicar_estado_inconsistente` · `listar_eventos_no_publicados` · `resumen_ventas_evento` · `verificar_vigencia_evento` |
| 🟩 `03-LLD.md` §4 (`:38-46`) y `06-Administrator-Guide.md` | **T1** `diagnosticar_publicacion` · **T2** `buscar_evento` · **T3** `estado_evento` · **T4** `listar_funciones` · **T5** `listar_tarifas_de_funcion` · **T6** `listar_valores_lookup` |

🟨 No es un problema de estilo. 🟩 El nombre de la tool es **contrato de tres puntas**: se registra en el `ToolRegistry` del tenant, lo emite el modelo en `tool_calls`, y lo rutea la API adaptadora. Tres nombres distintos para la misma capacidad significan que **el modelo va a invocar una tool que no existe**.

🟩 El plan de sprints ya lo declaró deuda bloqueante (`T-0.5`, 8 puntos, precondición del épico E-2).

### Decisión

**El catálogo canónico es el de `03-LLD.md` / `06-Administrator-Guide.md`: T1…T6, con los nombres exactos de la tabla siguiente. `01-SAD.md` §6.3 y `02-HLD.md` §12.1 se corrigen para reflejarlo. Los nombres son `snake_case`, en español, con verbo explícito. Ningún otro nombre existe.**

| ID | Nombre canónico | Entrada | Salida | Autorización | Fase |
|---|---|---|---|---|---|
| ⭐ **T1** | `diagnosticar_publicacion` | `idEvento` | `{publicado, pausado, activo, causa: CausaNoPublicado, eslabon, detalle, deepLink, advertencias[], evidencia[]}` | Evento ∈ alcance | **F1** |
| **T2** | `buscar_evento` | `texto?` \| `idEvento?` | `[{id, nombre, publicado, pausado, activo}]` | Alcance del usuario | **F1** |
| **T3** | `estado_evento` | `idEvento` | `{pausado, activo, publicado, esInconsistente}` | Evento ∈ alcance | **F1** |
| **T4** | `listar_funciones` | `idEvento` | `[{id, fecha, descripcion, activo, tieneUbicaciones}]` | Evento ∈ alcance | **F1** |
| **T5** | `listar_tarifas_de_funcion` | `idFuncion` | `[{idUbicacion, ubicacion, tarifas:[{id, descripcion, precio}]}]` | Función ∈ alcance | **F1** |
| **T6** | `listar_valores_lookup` | `catalogo` | `[{id, descripcion}]` | — | **F1** |

🟨 **Reglas de nomenclatura** (para que el próximo catálogo no vuelva a divergir):

| # | Regla | Ejemplo |
|---|---|---|
| N1 | `snake_case`, minúsculas, sin acentos | `listar_tarifas_de_funcion` |
| N2 | Verbo en infinitivo + objeto | `diagnosticar_publicacion` ✅ · `evento_estado` ❌ |
| N3 | El objeto usa el **vocabulario de la base**, no el de la UI | `..._publicacion` ✅ (F2: «publicado» es UI) |
| N4 | Sin abreviaturas | `listar_funciones` ✅ · `list_func` ❌ |
| N5 | Una tool = una capacidad. Sin `modo`/`accion` como parámetro | ❌ `consultar_evento(modo:"diagnostico")` |

🟨 **Tabla de migración** — nombres muertos y su destino. 🟩 `grep` de cualquiera de ellos en la documentación debe dar **0 hits** al cerrar `T-0.5`:

| Nombre muerto | Origen | Destino |
|---|---|---|
| `diagnosticar_evento` | 🟩 SAD §6.3 | → **T1** `diagnosticar_publicacion` |
| `listar_mis_eventos` | 🟩 SAD §6.3 | → **T2** `buscar_evento` (sin argumentos = mi alcance) |
| `detalle_funcion` | 🟩 SAD §6.3 | → **T5** `listar_tarifas_de_funcion` |
| `explicar_regla` | 🟩 SAD §6.3 | ❌ **No es tool: es RAG** ([ADR-006](#7-adr-006--arquitectura-de-conocimiento-híbrida-rag-para-lo-estable-tools-para-lo-volátil)). Ya lo decía el SAD |
| `detalle_configuracion_evento` | 🟩 HLD §12.1 | → **T4** + **T5** (se parte: devolvía el árbol entero) |
| `explicar_estado_inconsistente` | 🟩 HLD §12.1 | → **T3** `estado_evento` (`esInconsistente`) + T1 (`CausaNoPublicado.Inconsistente`) |
| `listar_eventos_no_publicados` | 🟩 HLD §12.1 | ⏸ Fase 2 (tenant admin) |
| `resumen_ventas_evento` | 🟩 HLD §12.1 | ❌ Fuera del MVP ([ADR-011](#12-adr-011--alcance-del-mvp-diagnosticar-la-cadena-eventofunciónfuncionubicaciontarifa)) |
| `verificar_vigencia_evento` | 🟩 HLD §12.1 | 🚫 **Bloqueada** ([ADR-012](#13-adr-012--stored-procedures-no-verificables-se-bloquea-la-capacidad-no-se-adivina)) |

```mermaid
flowchart LR
    subgraph SAD["🟩 01-SAD §6.3"]
        S1["diagnosticar_evento"]
        S2["listar_mis_eventos"]
        S3["detalle_funcion"]
        S4["explicar_regla"]
    end
    subgraph HLD["🟩 02-HLD §12.1"]
        H1["diagnosticar_publicacion"]
        H3["detalle_configuracion_evento"]
        H4["explicar_estado_inconsistente"]
        H7["verificar_vigencia_evento"]
    end
    subgraph CAN["⚖️ CANÓNICO — ADR-016"]
        T1["T1 diagnosticar_publicacion"]
        T2["T2 buscar_evento"]
        T3["T3 estado_evento"]
        T4["T4 listar_funciones"]
        T5["T5 listar_tarifas_de_funcion"]
        T6["T6 listar_valores_lookup"]
    end
    S1 --> T1
    H1 --> T1
    S2 --> T2
    S3 --> T5
    H3 --> T4
    H3 --> T5
    H4 --> T3
    S4 -.->|"❌ es RAG"| X["KB"]
    H7 -.->|"🚫 sprocs opacos"| B["bloqueada"]

    classDef can fill:#e6f4ea,stroke:#137333,stroke-width:2px,color:#000
    classDef dead fill:#f1f3f4,stroke:#9aa0a6,color:#000
    class T1,T2,T3,T4,T5,T6 can
    class S1,S2,S3,S4,H1,H3,H4,H7 dead
```

### Alternativas consideradas

| Alternativa | Atractivo | Por qué se descarta |
|---|---|---|
| **Canonizar el catálogo del SAD** (`diagnosticar_evento`, …) | 🟩 El SAD es el documento de mayor jerarquía y va primero en el orden de lectura. | 🟨 Es el catálogo **menos desarrollado**: 4 tools, una de las cuales 🟩 el propio SAD admite que **no es tool** (`explicar_regla`: *«Es RAG, no tool»*). 🟨 Y `diagnosticar_evento` viola N3: sugiere que diagnostica *el evento* en general, cuando 🟩 diagnostica **la publicación** (F3). |
| **Canonizar el catálogo del HLD** (7 tools) | 🟩 El más completo funcionalmente; ya tiene autorización y fases por tool. | 🟨 Mezcla F1 con F2 y con capacidades 🟩 bloqueadas (`verificar_vigencia_evento`) o 🟩 diferidas (`resumen_ventas_evento`): canonizarlo sería declarar contrato para lo que no vamos a construir. 🟨 Y `detalle_configuracion_evento` devuelve **el árbol entero** en una llamada — cómodo, pero 🟩 con el precio en la tabla puente el payload crece por función×ubicación×tarifa y **el LLM paga tokens por datos que no pidió**. Partirlo en T4+T5 permite que el modelo baje sólo el nivel que necesita. |
| **Inventar un catálogo nuevo** | Oportunidad de hacerlo bien de cero. | 🟨 Produciría un **cuarto** catálogo. 🟩 Y el del LLD ya es el mejor de los tres y es el que 🟩 `06-Administrator-Guide.md:823` y el plan de sprints (T-2.3) ya referencian. ❌ |
| **Nombres en inglés** (`diagnose_publication`) | 🟦 Convención habitual de APIs; consistente con `tool_call`. | 🟨 El dominio es español (`sys_VentaEntradas_Eventos`, `Pausado`, `Tarifas`) y el system prompt, la KB y los usuarios también. 🟨 Traducir sólo los nombres de tools agregaría una capa de traducción **dentro** del razonamiento del modelo, sin beneficio. |

### Consecuencias positivas

- 🟨 **`03-LLD.md` se puede escribir sin decidir nada.** Era el bloqueo: 🟩 el LLD tiene §4.1-§4.8 **sin cuerpo, sólo TOC**, esperando esta decisión.
- 🟨 **El catálogo cubre F1 completo y nada más.** Cada tool tiene un caso de uso del MVP; ninguna es especulativa.
- 🟨 **T4/T5 separadas siguen la cadena F1.** 🟩 La estructura del catálogo **es** el modelo de dominio: `listar_funciones` → `listar_tarifas_de_funcion` es literalmente `Evento→Función→FuncionUbicacion→Tarifa`. 🟨 El modelo aprende la relación **recorriéndola**, que es lo que el usuario pidió.
- 🟦 **N1-N5 hacen que el próximo catálogo no diverja** — la divergencia se originó en no tener reglas.

### Consecuencias negativas

- 🟨 **Hay que editar `01-SAD.md` §6.2/§6.3 y `02-HLD.md` §12.1**, más los endpoints del SAD (`:609-611`) que usan kebab-case (`/diagnosticar-evento`). 🟩 Trabajo declarado en `T-0.5`.
- 🟨 **Partir `detalle_configuracion_evento` en T4+T5 cuesta una vuelta de tool-loop más** para preguntas que necesitan el árbol completo. 🟩 Con el tope de 3 vueltas ([ADR-004](#5-adr-004--function-calling-genérico-en-iaconnect-no-un-hack-de-boletería)), `T1→T4→T5` consume el presupuesto entero. **Riesgo real**; mitigación: T1 ya devuelve el `eslabon`, así que el camino común no necesita T4/T5.
- 🟨 **T2 con y sin argumentos hace dos cosas** (buscar / listar mi alcance). Roza N5. Se acepta: 🟩 es una sola capacidad (*«mostrame eventos»*) con un filtro opcional.
- 🟨 **T3 se superpone parcialmente con T1**: T1 ya devuelve `pausado`/`activo`. Se conserva porque *«¿está publicado?»* no debe pagar el traversal completo.
- 🟨 **El nombre `listar_valores_lookup` es jerga técnica** que un LLM podría no asociar a *«¿qué tipos de evento hay?»*. Mitigación: la descripción del schema, no el nombre.

### Estado

🟨 **Propuesto.** ⚖️ **Supersede** `01-SAD.md` §6.3 y `02-HLD.md` §12.1. Depende de [ADR-011](#12-adr-011--alcance-del-mvp-diagnosticar-la-cadena-eventofunciónfuncionubicaciontarifa). Habilita [ADR-004](#5-adr-004--function-calling-genérico-en-iaconnect-no-un-hack-de-boletería), [ADR-006](#7-adr-006--arquitectura-de-conocimiento-híbrida-rag-para-lo-estable-tools-para-lo-volátil), [ADR-009](#10-adr-009--dos-tenants-por-perfil-de-usuario-no-un-system-prompt-condicional) y la redacción de `03-LLD.md` §4.

### Evidencia

| Afirmación | Fuente |
|---|---|
| 🟩 Catálogo del SAD (4 tools), con `explicar_regla` admitida como RAG | [`01-SAD.md:619-622`](01-SAD.md) §6.3 |
| 🟩 Endpoints del SAD en kebab-case | [`01-SAD.md:609-611`](01-SAD.md) §6.2 |
| 🟩 Catálogo del HLD (7 tools), con vigencia bloqueada y ventas diferida | [`02-HLD.md:1572-1578`](02-HLD.md) §12.1 |
| 🟩 Catálogo T1…T6 del LLD | [`03-LLD.md:38-46`](03-LLD.md) §4 · [`06-Administrator-Guide.md:823`](06-Administrator-Guide.md) |
| 🟩 La divergencia está verificada y declarada bloqueante | [`07-Plan-Sprints-Capacitacion.md:270`](07-Plan-Sprints-Capacitacion.md) T-0.5 |
| 🟩 `03-LLD.md` §4.1-§4.8 sin cuerpo, a la espera de esta decisión | [`03-LLD.md`](03-LLD.md) TOC |
| Precio en la tabla puente ⇒ el árbol completo crece por función×ubicación×tarifa | `SysTarifasUFuncionUbicacionModel.cs:17-19` |

---

## 18. ADR-017 — ⚖️ Nomenclatura canónica del enum: `CausaNoPublicado` (resuelve incoherencia **B**)

> ⚖️ **ADR de desempate.** 🟩 El enum tiene **tres nombres** según el documento. **Este ADR elige uno.**

### Contexto

🟩 Divergencia verificada y declarada:

| Documento | Nombre | Evidencia |
|---|---|---|
| `02-HLD.md` §12.3 | **`CausaCode`** | 🟩 `:1614-1624` |
| `03-LLD.md` §4.2 | **`CausaNoPublicado`** | 🟩 contrato de §4.2 |
| `01-SAD.md` §8.3 | *(sin nombre: nodos sueltos de un flowchart)* — `SinTarifaConPrecio`, `SinFuncionActiva`, … | 🟩 `:1051-1059` |

🟨 Y el problema es peor que un nombre: **los valores tampoco coinciden**. 🟩 El HLD dice `TarifasSinPrecio`; el SAD dice `SinTarifaConPrecio`. Son el mismo concepto con las palabras invertidas — lo más fácil de confundir y lo más difícil de detectar leyendo.

🟨 El enum no es un detalle interno: 🟩 es el contrato de salida de T1, el eje del `switch` de `DeepLinkBuilder` ([ADR-002](#3-adr-002--deep-links-devueltos-por-la-tool-jamás-construidos-por-el-llm)), la clave de las plantillas de degradación ([ADR-014](#15-adr-014--fallback-ante-proveedor-llm-caído-degradación-determinística-no-failover)) y la unidad de medida de CE-1 ([ADR-015](#16-adr-015--medición-del-éxito-y-criterio-de-continuidad-go--no-go)). 🟨 **Cuatro ADR dependen de que este tipo tenga un nombre.**

### Decisión

**El nombre canónico es `CausaNoPublicado`. Los valores canónicos son los siete de la tabla siguiente. `Ninguna` reemplaza a `OK`. `CausaCode` y los nodos del flowchart del SAD §8.3 quedan muertos.**

```csharp
// 🟨 PROPUESTA — BoleteriaCore.AI.Api/Contracts/CausaNoPublicado.cs
// ⚖️ ADR-017 · nombre y valores canónicos. NO renombrar sin supersedir este ADR.
public enum CausaNoPublicado
{
    Ninguna,             // 🟩 publicado y correcto (D4). Antes: "OK"
    TarifasSinPrecio,    // ⭐ 🟩 F3, el caso del 80% (:390-405). Antes: "SinTarifaConPrecio"
    SinFunciones,        // 🟩 el evento no tiene funciones
    FuncionesInactivas,  // 🟩 tiene funciones, ninguna activa. Antes: "SinFuncionActiva"
    SinUbicaciones,      // 🟩 funciones sin FuncionUbicacion
    Inconsistente,       // 🟩 Pausado=false Y Activo=false (F4, AccionPausar :441-461)
    Desconocida          // ⚠ ninguna de las anteriores ⇒ hand-off. JAMÁS se infiere
}
```

| Valor | Condición | Deep-link | Diálogo |
|---|---|---|---|
| `Ninguna` | 🟩 Publicado, con ≥1 función activa con precio | `/eventos/{slug}` | D4 |
| ⭐ `TarifasSinPrecio` | 🟩 Hay funciones activas; **ninguna** con `Precio > 0` | `?idFuncion={id}` | D1 |
| `SinFunciones` | 🟩 El evento no tiene funciones | `?idEvento=&idLugar=` | D3 |
| `FuncionesInactivas` | 🟩 Tiene funciones; **ninguna** activa | `/ParametrosEventosEdit?idEvento=` | D2 |
| `SinUbicaciones` | 🟩 Funciones sin `FuncionUbicacion` | `/ParametrosEventosEditLugares?idEvento=` | — |
| `Inconsistente` | 🟩 `Pausado=false` **y** `Activo=false` | `/ParametrosEventosEdit?idEvento=` | D6 |
| `Desconocida` | Ninguna de las anteriores | ❌ ninguno → hand-off | §8.2 |

### Alternativas consideradas

| Alternativa | Atractivo | Por qué se descarta |
|---|---|---|
| **`CausaCode`** (lo que dice 🟩 el HLD `:1614-1624`) | Es el nombre **más usado** en la documentación existente: 🟩 aparece en `02-HLD.md` §12.1, §12.2, §12.3, §13 y en el plan de sprints. Canonizarlo minimiza las ediciones. | 🟨 **Es un stutter bilingüe**: `Causa` + `Code` mezcla español e inglés para decir dos veces lo mismo — el tipo *es* el código, el sufijo no agrega información. 🟦 Convención: los enums no llevan sufijo `Code`/`Enum`/`Type` salvo que exista otro `Causa`. 🟨 Y `CausaCode` **no dice de qué es causa**: en un `switch` suelto, `CausaCode.SinFunciones` no se lee tan claro como `CausaNoPublicado.SinFunciones`. 🟨 «Requiere menos ediciones» es el peor criterio para elegir un contrato que va a vivir años. |
| **Los nodos del flowchart del SAD** (`SinTarifaConPrecio`, `SinFuncionActiva`) | Descriptivos y precisos; nombran la condición exacta. | 🟩 **Ni siquiera son un enum**: son nodos de un diagrama, sin tipo contenedor. 🟨 Y son **más largos sin ser más claros**: `SinTarifaConPrecio` obliga a parsear una doble negación (*sin* tarifa *con* precio) — justo lo que un desarrollador apurado lee mal. `TarifasSinPrecio` se lee de corrido. |
| **`MotivoNoPublicacion`** / **`EstadoPublicacion`** | Nombres nuevos, sin deuda; `EstadoPublicacion` describiría también `Ninguna`. | 🟨 Produce un **cuarto** nombre. 🟨 Y `EstadoPublicacion` es **peor semánticamente**: 🟩 «Publicado» no existe como estado en la base (F2) — nombrar un tipo `EstadoPublicacion` **reintroduciría en el código la ficción que la UI ya inventó**. El tipo debe nombrar *por qué no salió*, no fingir una máquina de estados inexistente. ❌ |
| **Conservar `OK` en vez de `Ninguna`** | 🟩 Es lo que dice el HLD. Universalmente entendido. | 🟨 `CausaNoPublicado.OK` **se lee como una contradicción**: la causa de no-publicación es… ¿OK? `CausaNoPublicado.Ninguna` se lee natural: *no hay causa de no publicación*. 🟨 Es exactamente el ajuste que el nombre canónico **exige**, y es la mejor prueba de que el nombre es bueno: **obliga a que los valores tengan sentido**. |

### Consecuencias positivas

- 🟨 **El tipo dice qué responde.** `CausaNoPublicado` responde *«¿por qué no se publicó?»* — que 🟩 es literalmente la pregunta del caso.
- 🟨 **`Ninguna` en vez de `OK` hace que el `switch` se lea como prosa**: `CausaNoPublicado.Ninguna ⇒ está todo bien`.
- 🟨 **Los valores quedan en positivo-legible**: `TarifasSinPrecio` en vez de `SinTarifaConPrecio` elimina la doble negación.
- 🟨 **Desbloquea cuatro ADR** (002, 005, 014, 015) y la redacción de `03-LLD.md` §4.2.
- 🟩 **Coincide con lo que ya está escrito en este mismo documento**: el snippet de `DeepLinkBuilder` en [ADR-002](#3-adr-002--deep-links-devueltos-por-la-tool-jamás-construidos-por-el-llm) ya usa `CausaNoPublicado`. Cero deuda interna.

### Consecuencias negativas

- 🟨 **Es el nombre menos usado hoy** ⇒ hay que editar `02-HLD.md` §12.1-§12.3, §13 y `07-Plan-Sprints-Capacitacion.md`. 🟩 Declarado en `T-0.5` con criterio de cierre: *«`grep` de los nombres viejos → 0 hits»*.
- 🟨 **`Ninguna` puede confundirse con «no sé»**, que es `Desconocida`. Mitigación: comentario en el enum + los dos son casos del fixture de [ADR-005](#6-adr-005--la-regla-de-publicación-se-reimplementa-en-la-tool-con-test-de-equivalencia).
- 🟩 **El enum está probablemente incompleto.** 🟨 El riesgo H7 del HLD lo dice: el flujo de **funciones ilimitadas** no fue analizado y puede tener reglas propias ⇒ `Desconocida` se va a disparar más de lo previsto. 🟨 Este ADR fija el **nombre**, no garantiza la **cobertura**: eso lo verifica el sprint 0.
- 🟨 **Un enum cerrado en el contrato de una API es rígido.** Agregar un valor es un cambio de contrato que impacta `DeepLinkBuilder`, las plantillas de degradación y la telemetría. 🟦 Es deliberado: **la rigidez es la feature** — un enum abierto invitaría a devolver strings y volveríamos a la prosa que [ADR-006](#7-adr-006--arquitectura-de-conocimiento-híbrida-rag-para-lo-estable-tools-para-lo-volátil) y `02-HLD.md` §12.2 prohíben.

### Estado

🟨 **Propuesto.** ⚖️ **Supersede** `02-HLD.md` §12.3 (`CausaCode`) y los nodos de `01-SAD.md` §8.3. Habilita [ADR-002](#3-adr-002--deep-links-devueltos-por-la-tool-jamás-construidos-por-el-llm), [ADR-005](#6-adr-005--la-regla-de-publicación-se-reimplementa-en-la-tool-con-test-de-equivalencia), [ADR-014](#15-adr-014--fallback-ante-proveedor-llm-caído-degradación-determinística-no-failover), [ADR-015](#16-adr-015--medición-del-éxito-y-criterio-de-continuidad-go--no-go).

### Evidencia

| Afirmación | Fuente |
|---|---|
| 🟩 El HLD lo llama `CausaCode` y define 7 valores | [`02-HLD.md:1614-1624`](02-HLD.md) §12.3 |
| 🟩 El LLD lo llama `CausaNoPublicado` | [`03-LLD.md`](03-LLD.md) §4.2 (contrato) |
| 🟩 El SAD usa nodos sueltos sin tipo (`SinTarifaConPrecio`, `SinFuncionActiva`) | [`01-SAD.md:1051-1059`](01-SAD.md) §8.3 |
| 🟩 La divergencia de tres nombres está verificada y declarada | [`07-Plan-Sprints-Capacitacion.md:200,1287`](07-Plan-Sprints-Capacitacion.md) |
| 🟩 `Desconocida` es valor de primera clase; sin él el LLM fuerza la categoría | [`02-HLD.md:1626-1627`](02-HLD.md) |
| 🟩 «Publicado» no existe como estado ⇒ no nombrar el tipo `EstadoPublicacion` | `ParametrosEventosEdit.razor.cs:174` · `SysVentaEntradasEventosModel.cs:57` |
| 🟩 `Inconsistente` es alcanzable vía `AccionPausar` | `ParametrosEventos.razor.cs:441-461` |
| 🟩 Riesgo H7: funciones ilimitadas ⇒ enum posiblemente incompleto | [`02-HLD.md:1792-1797`](02-HLD.md) |

---

## 19. Tabla resumen de ADRs

| ID | Título | Decisión en una línea | Estado |
|---|---|---|---|
| **001** | API adaptadora `BoleteriaCore.AI.Api` | Proyecto nuevo en la solución de BoleteriaCore que expone las tools por HTTP/JSON y reusa los DataManagers; IAConnect no toca la base del dominio | 🟨 Propuesto |
| **002** | Deep-links devueltos por la tool | `DeepLinkBuilder` arma la URL desde plantillas `const`; el LLM sólo transcribe el campo `deepLink` y el widget filtra por allowlist | 🟨 Propuesto |
| **003** | Propagación de identidad | JWT de vida corta por token-exchange de la cookie del BO; la API deriva el alcance del token y jamás de un parámetro | 🟨 Propuesto |
| **004** | Function-calling genérico | El tool-loop se construye en IAConnect como capacidad declarativa por tenant; cero conocimiento de boletería en el gateway | 🟨 Propuesto |
| ⭐ **005** | **Dónde vive la regla de publicación** | **Se reimplementa en `DiagnosticoPublicacionService` aceptando la duplicación, contenida por un test de equivalencia en CI contra el predicado del code-behind** | 🟨 Propuesto |
| **006** | RAG vs. tools vs. híbrido | Híbrido con frontera dura: lo que depende de una fila es tool; lo estable es RAG; sin tool disponible se deriva, nunca se rellena con RAG | 🟨 Propuesto |
| **007** | ¿Ejecuta o informa? | Sólo lectura en la v1: diagnostica y deriva al flujo nativo; permisos de BD de sólo lectura como barrera de infraestructura | 🟨 Propuesto |
| **008** | Widget | Componente Blazor en `MainLayout`, una línea de diff; no script de CDN ni iframe ni página dedicada | 🟨 Propuesto |
| **009** | Dos tenants por perfil | `organizador` y `admin`, cada uno con su prompt, KB y subconjunto de tools; el tenant lo elige el servidor | 🟨 Propuesto |
| ⚖️ **010** | **Tenant sobre un sistema sin multi-tenant** | **Sin sufijo de municipio: el tenant modela la audiencia; el aislamiento lo da `alcance(sub)` en la API — supersede `01-SAD.md` §6.6** | 🟨 Propuesto |
| **011** | Alcance del MVP | Diagnóstico de la cadena de 4 saltos en el Backoffice; fuera: portal, escritura, ventas, vigencia, descuentos, mapas, funciones ilimitadas | 🟨 Propuesto |
| **012** | Sprocs no verificables | Ninguna capacidad depende de un sproc no leído: se trae al repo, se reimplementa con test, o **se bloquea**. Inferir está prohibido | 🟨 Propuesto |
| **013** | Dueño y curaduría de la KB | Dueño funcional aprueba el contenido; KB versionada en Git y publicada por pipeline idempotente; edición manual sólo como emergencia auditada | 🟨 Propuesto |
| **014** | Fallback LLM caído | Degradación determinística (tool + plantilla por causa), no failover automático de proveedor | 🟨 Propuesto |
| **015** | Go / no-go | Dos métricas de decisión: resolución ≥ 40% y CE-1 ≥ 95%, a 3 meses, con umbrales pactados antes; «abandonar» es salida legítima | 🟨 Propuesto |
| ⚖️ **016** | **Catálogo canónico de tools** | **T1 `diagnosticar_publicacion` · T2 `buscar_evento` · T3 `estado_evento` · T4 `listar_funciones` · T5 `listar_tarifas_de_funcion` · T6 `listar_valores_lookup` — supersede `01-SAD.md` §6.3 y `02-HLD.md` §12.1** | 🟨 Propuesto |
| ⚖️ **017** | **Nombre canónico del enum** | **`CausaNoPublicado`, con `Ninguna` en lugar de `OK`; `CausaCode` y los nodos del SAD §8.3 quedan muertos — supersede `02-HLD.md` §12.3** | 🟨 Propuesto |

🟨 **Ninguna decisión está implementada.** Las tres ⚖️ son ejecutables de inmediato: son ediciones de documentación (`T-0.5`) y **bloquean** la escritura de `03-LLD.md` §4 y del épico E-2.

---

## 20. Trazabilidad de evidencia

🟩 = verificado en la fuente citada · 🟦 = práctica de industria · 🟨 = interpretación propia.

### 20.1 Hechos del dominio BoleteriaCore

| # | Afirmación | Marca | Fuente | ADR que la usa |
|---|---|---|---|---|
| 1 | `sys_Tarifas` **no tiene FK alguna**; la cadena real es `Evento 1—N Función 1—N FuncionUbicacion N—N Tarifa` | 🟩 | `SysTarifasModel.cs:11-33` | 001, 005, 011, 016 |
| 2 | El **precio vive en la tabla puente** `sys_Tarifas_U_FuncionUbicacion` (`Precio`, `Precio_Menores`) | 🟩 | `SysTarifasUFuncionUbicacionModel.cs:17-19` | 005, 011, 016 |
| 3 | «FuncionUbicacion es la tabla más importante del modelo» | 🟩 | [`ia-db/indexes/02_Modelo-Dominio.md`](../../../../../BD/Boleteria.Core.Documentacion/ia-db/indexes/02_Modelo-Dominio.md) | 011 |
| 4 | «Publicado» **no existe en la base**: es ViewModel que invierte `Pausado` | 🟩 | `ParametrosEventosEdit.razor.cs:174` | 005, 013, 016, 017 |
| 5 | `Activo` está mapeado; `Pausado` **no** está en el Model | 🟩 | `SysVentaEntradasEventosModel.cs:57` | 001, 005, 017 |
| 6 | `Pausado` se lee cruda del `DataRow` | 🟩 | `ParametrosEventos.razor.cs:194,472` | 001, 005 |
| 7 | La regla real: **≥1 tarifa con `Precio > 0` en función activa** | 🟩 | `ParametrosEventos.razor.cs:390-405` → modal `:422-436` | 005, 006, 008, 017 |
| 8 | Regla duplicada en la pantalla de edición | 🟩 | `ParametrosEventosEdit.razor.cs:1090-1105` → `:1165+` | 005 |
| 9 | Despublicación automática al desactivar la última función con precios | 🟩 | `ParametrosEventosEdit.razor.cs:1019-1034` → `:1149-1163` | 005 |
| 10 | ⚠ `AccionPausar` despausa **sin** validar tarifas; `AccionCambiarEstado` sí valida | 🟩 | `ParametrosEventos.razor.cs:441-461` vs. `:386-420` | 005, 007, 012, 017 |
| 11 | **Toda la validación es client-side**; sin Service ni excepción de dominio | 🟩 | grep exhaustivo sobre `Services/`, `Exceptions/` | 005, 007 |
| 12 | `UpdateByPausado` existe y es invocable directamente | 🟩 | `SysVentaEntradasEventosDataManager.cs:32-42` | 007, 012 |
| 13 | `lut_Parametros` es **clave-valor global** (`Codigo`, `Valor`, `Observaciones`), sin FK ni scope | 🟩 | `LutParametrosModel.cs:11-15` | 010, 011 |
| 14 | Ningún parámetro se valida como obligatorio antes de publicar | 🟩 | verdad de referencia §"lut_Parametros" | 011 |
| 15 | `ParametrosService` cachea en `IConfiguration` | 🟩 | `Services/ParametrosService.cs:11-65` | 010 |
| 16 | 🟨 «Parámetros» en el BO es el **módulo** de administración, no la tabla | 🟨 | `Components/Pages/Parametros/*` | 011 |
| 17 | **BoleteriaCore no es multi-tenant**; lo más cercano es `GP_IdMunicipio` y `CONFIG_codMunicipio` | 🟩 | `SysVentaEntradasEventosModel.cs:23` | 003, 010 |
| 18 | Que `GP_IdMunicipio` sea aislamiento es **inferencia, no hecho** | 🟨 | verdad de referencia §"No verificado" | 003, 010 |
| 19 | **Cuerpos de los sprocs ausentes**: sólo `issue-505.sql` e `issue-506.sql` | 🟩 | `DataManager/Migraciones/` | 005, 011, 012 |
| 20 | Sin DDL: FKs y cardinalidades **inferidas** de campos `Id_*` y JOINs | 🟩 | verdad de referencia §"No verificado" | 012 |
| 21 | **Sin proyecto de tests** en la solución | 🟩 | [`ia-db ADR-0008`](../../../../../BD/Boleteria.Core.Documentacion/boleteria-core-docs/docs/04-decisions/) | 005, 007 |
| 22 | `DataEntityCore` compone el sproc por convención y **bindea posicionalmente** | 🟩 | `Notions.Core.Utils.DataManager/DataEntityCore.cs:18-27,43-46` | 001, 012 |
| 23 | `DataEntityCore.Configure()` es singleton estático con **una sola** connection string | 🟩 | `IAConnect.API/Program.cs:22` + `DataEntityCore.cs:33-256` | 001, 007 |
| 24 | Capa de servicios **sin interfaces** (RA-12) ⇒ mockear es difícil | 🟩 | [`01-SAD.md`](01-SAD.md) §3.2 | 001 |
| 25 | Wizard de alta: **6.212 líneas**; pieza completa: 11.777 | 🟩 | `ParametrosEventosAlta.razor.cs` · [`01-SAD.md`](01-SAD.md) DR-1 | 001, 005, 007, 008 |
| 26 | Cobertura de lectura parcial del wizard (no leídas 1508-2719, 3440-6212) | 🟨 | verdad de referencia §"No verificado" | 005, 011 |
| 27 | **Funciones ilimitadas**: flujo no analizado, puede tener reglas propias | 🟨 | verdad de referencia §"No verificado" | 005, 011, 015, 017 |
| 28 | Descuentos y combos **no participan de la publicación** | 🟩 | verdad de referencia §"Tarifas" | 011 |
| 29 | Perfiles del BO (`parametros`, `hacienda`, `control-acceso`) vía `TienePerfil()` | 🟩 | `MainLayout.razor.cs:79` · [`ia-db/08_Seguridad.md`](../../../../../BD/Boleteria.Core.Documentacion/ia-db/indexes/08_Seguridad.md) | 001, 003, 008, 009 |
| 30 | Ítem «Mesa de Ayuda» existente en el sidebar | 🟩 | `MainLayout.razor:54` | 008, 014 |

### 20.2 Hechos de rutas y deep-links

| # | Afirmación | Marca | Fuente | ADR |
|---|---|---|---|---|
| 31 | Rutas planas con query string; sin parámetros de ruta (RA-2) | 🟩 | `ParametrosEventosEdit.razor:1` · [`routes-map.md`](../../../../../BD/Boleteria.Core.Documentacion/boleteria-core-docs/docs/pieces/boleteria-core-backoffice/routes-map.md) | 002 |
| 32 | ⚠ **Las rutas de edición no llevan id en la ruta** ⇒ el deep-link con `{id}` **exige código nuevo** | 🟩 | ídem #31 | 002 |
| 33 | `/ParametrosEventosEditFunciones` tiene **dos firmas incompatibles** (RA-3) | 🟩 | `ParametrosEventosEditFunciones.razor.cs:24,26,28` · `ParametrosEventosEdit.razor.cs:260,1065` | 002 |
| 34 | `ParametrosMapasCoordenadas` **no tiene `@page`** pero el wizard navega ahí | 🟩 | `ParametrosMapasCoordenadas.razor:1-3` · `ParametrosEventosAlta.razor.cs:3483-3487` | 002, 011 |
| 35 | `/hacienda-evento` es destino de redirect y **no existe** entre las 38 rutas | 🟩 | `AuthController.cs#L72` · [`routes-map.md`](../../../../../BD/Boleteria.Core.Documentacion/boleteria-core-docs/docs/pieces/boleteria-core-backoffice/routes-map.md) | 002 |
| 36 | Plantillas de deep-link verificadas | 🟩 | `ParametrosEventosEdit.razor.cs:260,1055-1083` | 002, 017 |
| 37 | Las rutas del BO se sirven bajo **PathBase** obligatorio (valor por ambiente **no verificado**) | 🟩 | [`routes-map.md`](../../../../../BD/Boleteria.Core.Documentacion/boleteria-core-docs/docs/pieces/boleteria-core-backoffice/routes-map.md) | 002, 008 |

### 20.3 Hechos de IAConnect

| # | Afirmación | Marca | Fuente | ADR |
|---|---|---|---|---|
| 38 | **Cero function-calling**: sin `tool_use`/`tool_choice`/`function_call` | 🟩 | grep exhaustivo · [`01-SAD.md`](01-SAD.md) §3.3 (RI-1) | 004, 014 |
| 39 | RAG **léxico TF-IDF top-5 sin threshold** | 🟩 | `IAConnect.Application/Services/RAGEngine.cs:34-127` | 002, 004, 006, 013, 015 |
| 40 | `VectorEmbedding = null` siempre; `SerializeEmbedding` es código muerto | 🟩 | `IAConnect.Application/Services/KnowledgeService.cs:75` | 006, 013 |
| 41 | `PromptBuilder` interpola **sin escapado** (RI-10 · OWASP LLM01/LLM02) | 🟩 | `IAConnect.Application/Services/PromptBuilder.cs:10-55` | 001, 002, 003, 006, 013, 015 |
| 42 | `lut_Tenants` es raíz del particionado: proveedor, modelo, temperatura y system prompt por tenant | 🟩 | `IAConnect/scripts/01_create_database.sql:31-53` | 001, 004, 009, 010, 013, 014 |

### 20.4 Decisiones previas del bloque (y sus contradicciones)

| # | Afirmación | Marca | Fuente | ADR |
|---|---|---|---|---|
| 43 | 🟩 El SAD declara tenants con sufijo `-{municipio}` — **contradicho por ADR-010** | 🟩 | [`01-SAD.md:718-721,1331`](01-SAD.md) §6.6 | ⚖️ 010 |
| 44 | 🟩 El SAD declara el catálogo `diagnosticar_evento` / `listar_mis_eventos` / `detalle_funcion` / `explicar_regla` — **superado por ADR-016** | 🟩 | [`01-SAD.md:619-622`](01-SAD.md) §6.3 | ⚖️ 016 |
| 45 | 🟩 El SAD admite que `explicar_regla` «es RAG, no tool» | 🟩 | [`01-SAD.md:622`](01-SAD.md) | 006, 016 |
| 46 | 🟩 El HLD declara un catálogo de 7 tools — **superado por ADR-016** | 🟩 | [`02-HLD.md:1572-1578`](02-HLD.md) §12.1 | ⚖️ 016 |
| 47 | 🟩 El LLD declara T1…T6 — **canonizado por ADR-016** | 🟩 | [`03-LLD.md:38-46`](03-LLD.md) §4 · [`06-Administrator-Guide.md:823`](06-Administrator-Guide.md) | ⚖️ 016 |
| 48 | 🟩 El enum tiene **tres nombres**: `CausaCode` (HLD), `CausaNoPublicado` (LLD), nodos sueltos (SAD §8.3) | 🟩 | `02-HLD.md:1614-1624` · `03-LLD.md` §4.2 · `01-SAD.md:1051-1059` | ⚖️ 017 |
| 49 | 🟩 Principio «enum + datos, nunca prosa» | 🟩 | [`02-HLD.md:1586-1612`](02-HLD.md) §12.2 | 006, 014, 017 |
| 50 | 🟩 Dos narrativas (organizador / admin) sobre el mismo contrato | 🟩 | [`02-HLD.md:1595-1596`](02-HLD.md) | 009, 010 |
| 51 | 🟩 `verificar_vigencia_evento` **bloqueada por evidencia, no por esfuerzo** | 🟩 | [`02-HLD.md:1580-1584`](02-HLD.md) | 011, 012, 016 |
| 52 | 🟩 «Un booleano sin porqué es, otra vez, el modal» | 🟩 | [`02-HLD.md:1584`](02-HLD.md) | 012 |
| 53 | 🟩 `Desconocida` es valor de primera clase | 🟩 | [`02-HLD.md:1624-1627`](02-HLD.md) | 015, 017 |
| 54 | 🟩 CE-1: `CausaCode` == causa real, ≥ 95%, muestra auditada | 🟩 | [`02-HLD.md:1679`](02-HLD.md) §13 | 015 |
| 55 | 🟩 Riesgo H7: funciones ilimitadas ⇒ enum incompleto | 🟩 | [`02-HLD.md:1792-1797`](02-HLD.md) | 011, 015, 017 |
| 56 | 🟩 `T-0.5` declara la deuda documental (enum + catálogo) **bloqueante de E-2**, 8 puntos | 🟩 | [`07-Plan-Sprints-Capacitacion.md:200,270,1287`](07-Plan-Sprints-Capacitacion.md) | ⚖️ 010, 016, 017 |
| 57 | 🟩 `DiagnosticoPublicacionService` «reimplementa el LINQ de `:394-398`» | 🟩 | [`07-Plan-Sprints-Capacitacion.md:298`](07-Plan-Sprints-Capacitacion.md) T-2.3 | 005 |
| 58 | 🟩 `03-LLD.md` §4.1-§4.8 sin cuerpo, a la espera de estas decisiones | 🟩 | [`03-LLD.md`](03-LLD.md) TOC | 016, 017 |
| 59 | 🟩 Instrucción de no listar lookups en la KB (los trae T6) | 🟩 | [`06-Administrator-Guide.md:823`](06-Administrator-Guide.md) | 006, 013, 016 |

### 20.5 Alineación con el bloque hermano y el marco conceptual

| # | Afirmación | Marca | Fuente | ADR |
|---|---|---|---|---|
| 60 | GDA-Turnos decide **tools en el gateway** — misma decisión, otro consumidor | 🟩 | [`../GDA-Turnos/04-ADR.md`](../GDA-Turnos/04-ADR.md) ADR-006 | 004 |
| 61 | GDA-Turnos decide **RAG para lo estable, tools para lo volátil** | 🟩 | [`../GDA-Turnos/04-ADR.md`](../GDA-Turnos/04-ADR.md) ADR-004 | 006 |
| 62 | GDA-Turnos nombra tenants **por perfil** (`-ciudadano` / `-funcionario`), no por jurisdicción | 🟩 | [`../GDA-Turnos/04-ADR.md`](../GDA-Turnos/04-ADR.md) ADR-002 | 009, 010 |
| 63 | Marco de marcas 🟩/🟦/🟨 y criterio de evidencia | 🟩 | [`../Antecedentes/Analisis-Asistencia-IA-ChatBotIA.md`](../Antecedentes/Analisis-Asistencia-IA-ChatBotIA.md) bloques A-G | todos |
| 64 | Patrón de deep-link y disclosure de alcance | 🟩 | [`../Antecedentes/IA-Mercado-Libre.md`](../Antecedentes/IA-Mercado-Libre.md) | 002, 007 |
| 65 | Metodología transversal del gateway (no repetida acá) | 🟩 | [`../Ng-IAServices/01-SAD.md`](../Ng-IAServices/01-SAD.md) … [`06-Administrator-Guide.md`](../Ng-IAServices/06-Administrator-Guide.md) | todos |

### 20.6 Límites de esta evidencia

🟨 Se declaran explícitamente, porque un ADR que no dice qué no sabe es propaganda:

| Límite | Impacto sobre las decisiones |
|---|---|
| 🟩 **Cuerpos de sprocs ausentes** | ADR-005 puede diagnosticar sobre datos filtrados por reglas invisibles. ADR-012 acota; **no elimina**. |
| 🟩 **Sin DDL**: FKs y cardinalidades inferidas | El traversal de T1 asume relaciones no confirmadas físicamente. |
| 🟨 **Wizard leído parcialmente** (~1.800 de 6.212 líneas) | Puede haber validaciones de publicación no contempladas ⇒ enum incompleto (ADR-017). |
| 🟨 **Funciones ilimitadas sin analizar** | Fuente probable de `Desconocida` (ADR-011, ADR-015). |
| 🟨 **`GP_IdMunicipio` como criterio de alcance: inferencia** | ADR-003 diseña `alcance(sub)` sobre un supuesto. **Primera pregunta al responsable funcional.** |
| 🟩 **PathBase por ambiente no verificado** | ADR-002/008: el prefijo se resuelve en runtime; sin confirmar en producción. |
| 🟨 **Carga concurrente del BO no verificada** | ADR-008: el widget vive en el circuito; impacto de memoria no dimensionado. |
| 🟨 **Umbrales de ADR-015 sin respaldo empírico** | 40% / 95% / 15% son juicio. **Requieren acuerdo formal antes del despliegue.** |
| 🟨 **Circuit breaker en IAConnect: no verificado** | ADR-014 asume que se construye. |

---

> **Fin del documento.** 🟨 17 ADR, ninguno implementado. Las tres decisiones ⚖️ (010, 016, 017) son precondición de [`03-LLD.md`](03-LLD.md) §4 y de [`05-Operations-Guide.md`](05-Operations-Guide.md), que se escriben después de este documento y **lo implementan**.
