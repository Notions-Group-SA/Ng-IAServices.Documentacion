---
doc_id: GLOSSARY-001
doc_type: glossary
title: Glosario — IAConnect
version: 1.0.0
status: draft
origin: reverse-engineered
confidence: high
owner: pendiente-asignacion
last_review: 2026-07-15
review_cycle_days: 180
audience: [dev, qa, ops, arquitectos, agentes-automaticos]
classification: uso-interno
---

# Glosario — IAConnect

> Vocabulario controlado y normativo (Marco §12.3). Humanos y agentes usan estos términos canónicos; los
> sinónimos se registran como `alias`. Ante duda terminológica, este glosario decide.

| Término canónico | Alias | Definición |
|---|---|---|
| **Tenant** | cliente, organización, inquilino | Unidad de aislamiento de la solución: cada tenant tiene su proveedor IA, modelo, *system prompt*, límites, usuarios, sesiones y base de conocimiento propios. Tabla `lut_Tenants`. |
| **Proveedor IA** | provider, LLM provider | Servicio externo de modelos de lenguaje al que se enruta la solicitud: `Gemini`, `Claude` u `OpenAI`. Enum `ProveedorIA`. |
| **AIProviderFactory** | factory de proveedores | Componente que resuelve el `IAIProvider` concreto según `Proveedor_IA` del tenant. |
| **DataManager** | — | Componente de acceso a datos por tabla que invoca *stored procedures*. Patrón **DataEntity-DataManager**. |
| **DataEntityCore** | — | Núcleo de acceso a datos que centraliza la conexión SQL; se configura al arrancar con la cadena `IAConnect`. |
| **Sesión** | session, conversación | Hilo de conversación de un usuario externo dentro de un tenant. Tabla `sys_Sesiones`. |
| **Mensaje** | turno, message | Intervención en una sesión con rol `user`/`assistant`/`system`. Tabla `sys_Mensajes`. |
| **Fragmento de conocimiento** | chunk, fragmento RAG | Porción de un documento cargado, con su *embedding*, usada para RAG. Tabla `sys_Fragmentos_Conocimiento`. |
| **RAG** | retrieval-augmented generation | Técnica que recupera fragmentos relevantes del tenant y los inyecta en el prompt antes de llamar al proveedor IA. Componente `RAGEngine`. |
| **Embedding** | vector, `Vector_Embedding` | Representación vectorial de un fragmento para búsqueda por similitud. Columna `varbinary(MAX)`. |
| **PromptBuilder** | — | Componente que ensambla el prompt final (system prompt del tenant + contexto RAG + historial + mensaje). |
| **System prompt** | prompt de sistema | Instrucción base del tenant que condiciona al modelo. Columna `lut_Tenants.System_Prompt`. |
| **Completion** | completado | Generación de texto a partir de un prompt sin historial conversacional. Endpoint `/completion`. |
| **Analyze** | análisis | Procesamiento que clasifica un texto (sentimiento/entidades/categorización). Enum `TipoAnalisis`. |
| **Summarize** | resumen | Generación de un resumen de un texto. Endpoint `/summarize`. |
| **Improve** | mejora | Reescritura de un texto según un objetivo (`ObjetivoMejora`: gramática/claridad/formal/conciso). |
| **Métrica de uso** | métrica, usage metric | Registro de consumo por solicitud (tokens, proveedor, modelo, duración). Tabla `sys_Metricas_Uso`. |
| **Access token** | JWT, token de acceso | Token JWT de vida corta que autentica cada solicitud. Expira según config del tenant. |
| **Refresh token** | token de refresco | Token de vida larga para renovar el access token; revocable. Tabla `sys_Refresh_Tokens`. |
| **TenantResolverMiddleware** | — | Middleware que resuelve el tenant del contexto de la solicitud. |
| **TenantAccessFilter** | filtro de acceso al tenant | Filtro que valida que el usuario autenticado pueda operar sobre el `{tenantId}` de la ruta. |
| **Usuario** | operador/admin | Cuenta interna que administra u opera la solución. Roles `admin`/`operador`. Tabla `sys_Usuarios`. |
| **Widget de chat** | ChatWidget, IAConnectChat | Componente Blazor embebible que consume la API para ofrecer chat. Proyecto `IAConnect.ChatWidget`. |

## Convenciones de identificadores (§12.3)

| Prefijo | Significado | Dónde |
|---|---|---|
| `RF-` / `RNF-` | Requisito funcional / no funcional | `02-requirements/` |
| `ADR-` | Decisión de arquitectura | `04-decisions/` |
| `TC-` | Caso de prueba derivado del modelo | `03-data/test-data-matrix.md` |
| `CU-` | Caso de uso (del origen) | `docs/02_especificacion_funcional/` (origen) |
| `GAP-` | Brecha documental declarada | `docs-manifest.yaml` |
