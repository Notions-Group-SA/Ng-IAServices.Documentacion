---
doc_id: PIECE-INFRA-001
doc_type: piece-readme
title: IAConnect.Infrastructure — Acceso a datos y proveedores IA
version: 1.0.0
status: draft
origin: reverse-engineered
confidence: high
owner: pendiente-asignacion
last_review: 2026-07-15
review_cycle_days: 180
audience: [dev, arquitectos, agentes-automaticos]
classification: uso-interno
traces: []
supersedes: null
---

# IAConnect.Infrastructure

> `type: library` · .NET 8 (`net8.0`, `ImplicitUsings`, `Nullable` enable) — ver `IAConnect.Infrastructure/IAConnect.Infrastructure.csproj#L17-L21`.
> Documento generado por ingeniería inversa (solo lectura del código). Todo lo marcado como *(inferido)* no está explícito en el código de esta pieza.

## 1. Resumen ejecutivo

`IAConnect.Infrastructure` es la capa de infraestructura del gateway multi-tenant de IA. Cumple dos responsabilidades independientes:

1. **Acceso a datos** mediante un patrón propietario **DataEntity–DataManager** sobre **stored procedures de SQL Server**. Un único núcleo (`DataEntityCore`) ejecuta SPs por convención de nombre y mapea los resultados a *Models* por reflexión. Cada una de las 7 tablas tiene un par `Abstract` (armado de parámetros + índices) y `DataManager` (mapeo Model↔Entidad de dominio, implementa la interfaz del dominio).
2. **Proveedores IA**: una abstracción `IAIProvider` con tres implementaciones (`ClaudeProvider`, `GeminiProvider`, `OpenAIProvider`) y una fábrica `AIProviderFactory` (Factory + Strategy) que instancia el proveedor correcto según el campo `ProveedorIA` del *tenant* y desencripta su API key.

Dependencias declaradas (`IAConnect.Infrastructure.csproj#L3-L15`): `ProjectReference` a `IAConnect.Domain` y `IAConnect.Application`; paquetes `Microsoft.Data.SqlClient 5.2.*`, `Microsoft.Extensions.Http 8.*`, `Google_GenerativeAI 3.6.4`, `OpenAI 2.10.0`.

> ⚠️ Divergencia de capas *(inferido)*: Infrastructure referencia a `IAConnect.Application` (`csproj#L5`), lo cual invierte la dependencia habitual Application→Infrastructure. Ver §7.

---

## 2. Patrón DataEntity–DataManager

### 2.1 Cómo funciona `DataEntityCore`

`DataAccess/DataEntityCore.cs` es el núcleo compartido por todos los DataManagers. Puntos clave:

- **Configuración de conexión global y estática.** El connection string se guarda en un campo `static string? _connectionString` (`#L16`) y se fija **una sola vez al arranque** con `DataEntityCore.Configure(connectionString)` (`#L26-L29`). Si no se configuró, `CreateConnection()` lanza `InvalidOperationException` (`#L183-L189`). La configuración real la hace la API en el arranque: `DataEntityCore.Configure(builder.Configuration.GetConnectionString("IAConnect")!)` (`IAConnect.API/Program.cs#L22`, externo a esta pieza).
- **Convención de SP.** Cada instancia se crea con el nombre de tabla (`new("lut_Tenants")`, etc.) y compone el nombre del SP como `SP_{tabla}_{operación}`:

  | Método `DataEntityCore` | SP invocado | Traza |
  |---|---|---|
  | `AddAsync` | `SP_{tabla}_Add` | `#L35` |
  | `UpdateAsync` | `SP_{tabla}_Update` | `#L44` |
  | `DeleteAsync` | `SP_{tabla}_Delete` | `#L52` |
  | `GetAllAsync` / `GetListAllAsync<T>` | `SP_{tabla}_GetAll` | `#L64`, `#L75` |
  | `GetOneAsync<T>` | `SP_{tabla}_GetOne` | `#L88` |
  | `GetByAsync` / `GetListByAsync<T>` | `SP_{tabla}_GetBy_{indice}` | `#L102`, `#L113` |
  | `GetByCantidadAsync` | `SP_{tabla}_GetBy_{indice}_Cantidad` | `#L122` |

- **Ejecución centralizada.** `ExecuteAsync<T>` (`#L137-L181`) gestiona el ciclo de vida de conexión/transacción: si recibe una `SqlTransaction` reutiliza su conexión y NO la cierra; si no, crea y **descarta una conexión por llamada** (`ownConnection`, `#L143-L155`, `#L176-L180`). Crea un `SqlCommand` con `CommandType.StoredProcedure` (`#L162-L165`) y llama a **`SqlCommandBuilder.DeriveParameters(cmd)`** (`#L169`), que consulta a la BD los metadatos de parámetros del SP en tiempo de ejecución (round-trip extra por llamada — ver §7).
- **Mapeo posicional de parámetros.** `SetParameters` (`#L195-L205`) asigna el array `arParams` a los parámetros del SP **por posición** (saltando `@RETURN_VALUE`). Elementos sobrantes se ignoran silenciosamente (de ahí el `0` reservado al final de cada `InsertAsync`).
- **Mapeo de resultados a Models por reflexión.** `MapReaderToList<T>` (`#L221-L256`) empareja columnas del `SqlDataReader` con propiedades públicas escribibles por nombre (case-insensitive), maneja `DBNull` y convierte tipos con `Convert.ChangeType` / `Nullable.GetUnderlyingType`. También expone variantes que devuelven `DataSet` vía `SqlDataAdapter` (`#L62-L71`, `#L100-L109`).

### 2.2 Estructura Abstract + DataManager por tabla

```mermaid
flowchart LR
    subgraph Dominio["IAConnect.Domain"]
        IF["IXxxDataManager (interfaz)"]
        ENT["Entidad de dominio (p.ej. Tenant)"]
    end
    subgraph Infra["IAConnect.Infrastructure"]
        DM["XxxDataManager\n(mapea Model↔Entidad,\nimplementa la interfaz)"]
        ABS["XxxAbstract\n(arma object[] arParams,\nmétodos por índice)"]
        MODEL["XxxModel\n(POCO espejo de columnas)"]
        CORE["DataEntityCore\n(SP_{tabla}_{op}, mapeo por reflexión)"]
    end
    DB[("SQL Server\nStored Procedures")]

    DM -->|hereda| ABS
    DM -.->|implementa| IF
    DM -->|MapToEntity / MapToModel| ENT
    ABS -->|new(tabla) + arParams| CORE
    ABS --> MODEL
    CORE -->|MapReaderToList → Model| MODEL
    CORE -->|ExecuteReader / NonQuery / Scalar| DB
```

- **`XxxAbstract`** (`abstract class`): instancia `protected readonly DataEntityCore _dbManager = new("<tabla>")` (p.ej. `LutTenantsAbstract.cs#L11`), fija `CultureInfo.CurrentCulture = es-ES` en el ctor, y por cada operación arma el `object[] arParams` en el orden exacto que espera el SP. `Insert` **excluye** columnas gestionadas por la BD (`Id` IDENTITY, `Fecha_Alta`, `Fecha_Modificacion`, `Usuario_Alta`, `Usuario_Modificacion`) y agrega un `0` reservado al final (p.ej. `LutTenantsAbstract.cs#L27-L44`). `Update` antepone la PK (`#L53-L69`). Define métodos por índice: `GetBy_/GetListBy_/GetBy_*_Cantidad`.
- **`XxxDataManager`** (`class`): hereda del `Abstract`, implementa la interfaz del dominio de forma explícita y traduce entre el *Model* (espejo de columnas, nombres con guiones bajos) y la *Entidad* de dominio (nombres PascalCase) mediante `MapToEntity` / `MapToModel` (p.ej. `LutTenantsDataManager.cs#L11-L55`). Se registra en DI como `Scoped` (`IAConnect.API/Program.cs#L91-L97`).

### 2.3 DataManager → tabla → SPs → Model

| DataManager (namespace) | Tabla (`_dbManager`) | Model | Índices `GetBy_*` (además de CRUD) | Traza Abstract |
|---|---|---|---|---|
| `LutTenants` | `lut_Tenants` | `LutTenantsModel` (PK `Id_Tenant` string, **no** IDENTITY) | `Proveedor_IA`, `Activo` | `LutTenantsAbstract.cs#L11,L97-L137` |
| `SysUsuarios` | `sys_Usuarios` | `SysUsuariosModel` (PK `Id` int IDENTITY) | `Nombre_Usuario`, `Id_Tenant`, `Rol`, `Activo` | `SysUsuariosAbstract.cs#L11,L85-L171` |
| `SysSesiones` | `sys_Sesiones` | `SysSesionesModel` (PK `Id` int; `Id_Sesion` GUID) | `Id_Sesion`, `Id_Tenant`, `Activo`, `Id_Tenant_Activo` (compuesto) | `SysSesionesAbstract.cs#L11,L79-L165` |
| `SysMensajes` | `sys_Mensajes` | `SysMensajesModel` (PK `Id` long) | `Id_Sesion` | `SysMensajesAbstract.cs#L11,L85-L105` |
| `SysFragmentosConocimiento` | `sys_Fragmentos_Conocimiento` | `SysFragmentosConocimientoModel` (PK `Id` long; `Vector_Embedding` byte[]) | `Id_Tenant`, `Id_Tenant_Documento_Origen` (compuesto) | `SysFragmentosConocimientoAbstract.cs#L11,L77-L119` |
| `SysMetricasUso` | `sys_Metricas_Uso` | `SysMetricasUsoModel` (PK `Id` long) | `Id_Tenant`, `Id_Sesion`, `Fecha_Solicitud`, `Id_Tenant_Proveedor` (compuesto) | `SysMetricasUsoAbstract.cs#L11,L87-L173` |
| `SysRefreshTokens` | `sys_Refresh_Tokens` | `SysRefreshTokensModel` (PK `Id` int) | `Id_Usuario`, `Token`, `Revocado` | `SysRefreshTokensAbstract.cs#L11,L77-L139` |

Ejemplo de SPs concretos derivados de la convención para `lut_Tenants`: `SP_lut_Tenants_Add`, `SP_lut_Tenants_Update`, `SP_lut_Tenants_Delete`, `SP_lut_Tenants_GetAll`, `SP_lut_Tenants_GetOne`, `SP_lut_Tenants_GetBy_Proveedor_IA`, `SP_lut_Tenants_GetBy_Activo_Cantidad`.

> Nota: las 7 tablas siguen el mismo patrón; se leyó `DataEntityCore` completo, el par `LutTenants` completo y se verificaron los 6 restantes (Abstract + Model). No hay divergencias de patrón entre ellos.

---

## 3. Proveedores IA

### 3.1 Abstracción y fábrica

`AIProviderFactory` (`Providers/AIProviderFactory.cs`) implementa `IAIProviderFactory` (Domain) y aplica **Factory + Strategy**: recibe una `Tenant`, desencripta su API key y selecciona la implementación por `tenant.ProveedorIA.ToLower()` con un `switch` (`#L21-L31`):

```mermaid
flowchart TD
    Caller["Capa Application"] -->|CreateProvider(tenant)| F["AIProviderFactory"]
    F -->|DecryptApiKey(tenant.ApiKeyIA)| K["AES-256-CBC / env var\nIACONNECT_ENCRYPTION_KEY"]
    F -->|switch tenant.ProveedorIA.ToLower| SW{"proveedor"}
    SW -->|'claude'| C["ClaudeProvider\n(HttpClient nombrado 'Claude')"]
    SW -->|'gemini'| G["GeminiProvider\n(SDK Google_GenerativeAI)"]
    SW -->|'openai'| O["OpenAIProvider\n(SDK OpenAI)"]
    SW -->|otro| EX["ArgumentException\n'Proveedor no soportado'"]
    C & G & O -.->|implementan| IAIP["IAIProvider\n(ChatAsync, CompleteAsync,\nAnalyzeAsync, SummarizeAsync, ImproveAsync)"]
```

Cada proveedor recibe del *tenant* el **modelo, temperatura y máximo de tokens** (`AIProviderFactory.cs#L22-L28`): `NombreModelo`, `Temperatura`, `MaxTokens`. La API key desencriptada se pasa por constructor. Todos implementan las 5 operaciones de `IAIProvider` y devuelven un `AIResponse` con `Response`, `PromptTokens`, `CompletionTokens`, `Provider`.

### 3.2 Tabla de proveedores

| Proveedor (`ProveedorIA`) | Servicio externo | Endpoint base (sin credenciales) | Cliente / SDK | Autenticación (genérica) | Modelo por defecto declarado en `appsettings` |
|---|---|---|---|---|---|
| `claude` | Anthropic Messages API | `https://api.anthropic.com/` + ruta relativa `POST v1/messages` | `HttpClient` nombrado `"Claude"` + `System.Text.Json` | Header `x-api-key` con la key + header `anthropic-version: 2023-06-01` | `claude-3-sonnet-20240229` |
| `gemini` | Google Generative AI | Gestionado por el SDK `Google_GenerativeAI` (host `generativelanguage.googleapis.com` — *inferido*, no aparece en el código) | `GoogleAi` / `GenerativeModel` (SDK) | API key pasada al ctor `new GoogleAi(apiKey)` | `gemini-2.5-flash` |
| `openai` | OpenAI Chat Completions | Gestionado por el SDK `OpenAI` (host `api.openai.com` — *inferido*, no aparece en el código) | `ChatClient` (SDK) | API key pasada al ctor `new ChatClient(model, apiKey)` | `gpt-4` |

Trazas: base address Claude en `IAConnect.API/Program.cs#L81-L85` (cliente nombrado, `Timeout = 60s`); ruta `v1/messages` y headers en `ClaudeProvider.cs#L193-L196`; Gemini SDK en `GeminiProvider.cs#L18-L23`; OpenAI SDK en `OpenAIProvider.cs#L17-L22`; modelos por defecto en `IAConnect.API/appsettings.json#L25-L38`.

> ⚠️ **Contradicción reportada (gana el código).** El bloque `AIProviders:*` de `appsettings.json` (ApiKey + DefaultModel por proveedor) **NO es consumido por Infrastructure**. El modelo efectivo en runtime proviene de `tenant.NombreModelo` (registro de `lut_Tenants`), no del `DefaultModel` de config. Los valores de la columna "por defecto declarado" son informativos, no operativos. Ver §7.

### 3.3 Construcción de request (resumen por proveedor)

- **Claude** (`ClaudeProvider.cs`): arma `messages` (historial + mensaje de usuario, con bloque de imagen base64 si aplica, `#L120-L173`) y un `payload` con `model`, `max_tokens`, `temperature`, `system`, `messages` (`#L175-L185`); serializa en snake_case (`#L22-L26`). Parseo de respuesta: `content[0].text` y `usage.input_tokens` / `usage.output_tokens` (`#L218-L235`). Detección de MIME de imagen por prefijo base64 (`#L245-L251`).
- **Gemini** (`GeminiProvider.cs`): construye `GenerateContentRequest` con `SystemInstruction`, `GenerationConfig` (Temperature, MaxOutputTokens) y `Contents` (rol `assistant`→`model`), imágenes vía `InlineData/Blob` (`#L130-L180`). Tokens desde `UsageMetadata` (`#L182-L191`).
- **OpenAI** (`OpenAIProvider.cs`): construye `List<ChatMessage>` (System/Assistant/User, imagen vía `ChatMessageContentPart.CreateImagePart`, `#L142-L188`) y `ChatCompletionOptions` (MaxOutputTokenCount, Temperature, `#L190-L197`). Tokens desde `completion.Usage` (`#L199-L208`).

En los tres, si `Temperature`/`MaxTokens` del request son `> 0` se usan; si no, caen a los valores del tenant inyectados por constructor (p.ej. `ClaudeProvider.cs#L180-L181`).

---

## 4. Manejo de errores

- **Excepción de dominio unificada:** `ProviderUnavailableException` (`IAConnect.Domain/Exceptions/ProviderUnavailableException.cs#L3-L18`) lleva la propiedad `Provider` y un mensaje. Los tres proveedores envuelven **cualquier** excepción no-`ProviderUnavailableException` en ella (re-lanzando la propia sin re-envolver): patrón `catch (ProviderUnavailableException) { throw; } catch (Exception ex) { throw new ProviderUnavailableException(...) }` en cada método (p.ej. `ClaudeProvider.cs#L46-L50`, `OpenAIProvider.cs#L38-L48` que además captura `ClientResultException` del SDK).
- **Reintentos con backoff exponencial (Claude y Gemini):** `MaxRetries = 3` (`ClaudeProvider.cs#L20`, `GeminiProvider.cs#L15`), espera `2^(retries-1)` segundos (`ClaudeProvider.cs#L208`, `GeminiProvider.cs#L207`).
  - Claude reintenta ante códigos HTTP transitorios: `429`, `503`, `502`, `504` (`IsTransientStatusCode`, `#L237-L243`); si se agotan los reintentos o el error no es transitorio, lanza `ProviderUnavailableException` con `"Claude API error {status}: {body}"` (`#L212-L214`).
  - Gemini detecta transitorios por **texto del mensaje** de la excepción del SDK: contiene `"429"`, `"503"`, `"rate"` o `"unavailable"` (`IsTransientError`, `#L212-L218`) — heurística frágil (ver §7).
- **OpenAI:** **no** implementa bucle de reintentos propio; delega en el SDK y traduce `ClientResultException`/`Exception` a `ProviderUnavailableException` (`OpenAIProvider.cs#L38-L48`).
- **Timeouts:** solo Claude tiene timeout explícito, vía el `HttpClient` nombrado `"Claude"` con `Timeout = 60s` (`IAConnect.API/Program.cs#L84`, externo a la pieza). Gemini/OpenAI dependen del timeout por defecto de sus SDKs *(inferido)*.
- **Acceso a datos:** `DataEntityCore` no captura excepciones SQL; se propagan al llamador. La única excepción propia es `InvalidOperationException` si no se llamó a `Configure` (`DataEntityCore.cs#L185-L188`).

---

## 5. Configuración requerida

Esta pieza **no lee `IConfiguration` directamente**; consume configuración de forma indirecta a través del arranque de la API y de una variable de entorno. Claves relevantes (sin valores):

| Clave / fuente | Consumida por | Cómo |
|---|---|---|
| `ConnectionStrings:IAConnect` | `DataEntityCore` (toda la capa de datos) | La API la pasa a `DataEntityCore.Configure(...)` al arranque (`IAConnect.API/Program.cs#L22`). |
| Variable de entorno `IACONNECT_ENCRYPTION_KEY` (Base64) | `AIProviderFactory.DecryptApiKey` | Clave AES-256 para desencriptar `ApiKey_IA` del tenant (`AIProviderFactory.cs#L35-L47`). |
| Base address del `HttpClient` `"Claude"` | `ClaudeProvider` | Registrada por la API como cliente nombrado (`IAConnect.API/Program.cs#L81-L85`). |
| `AIProviders:*` (`appsettings.json#L25-L38`) | **Declarada pero NO consumida por Infrastructure** | El modelo/API key operativos vienen del tenant en BD, no de aquí. Reportado como gap (§7). |
| `Encryption:AesKey` (`appsettings.json#L22-L24`) | **Declarada pero NO consumida** | El código usa la variable de entorno `IACONNECT_ENCRYPTION_KEY`, no esta clave de config. Divergencia (§7). |

---

## 6. Snippets representativos

**§13-A — Firma del núcleo de datos (convención de SP + ejecución).** Fuente: `DataAccess/DataEntityCore.cs#L33-L44`, `#L137-L169`.

```csharp
public class DataEntityCore
{
    public DataEntityCore(string tableName) { /* … */ }
    public static void Configure(string connectionString) { /* … */ }

    public virtual async Task<int> AddAsync(object[] arParams, SqlTransaction? sqlTransaction = null)
        => await ExecuteAsync($"SP_{_tableName}_Add", arParams, sqlTransaction, /* … ExecuteScalar → int */);

    public virtual async Task<List<T>> GetListByAsync<T>(string indexName, object[] arParams) where T : new()
        => await ExecuteAsync($"SP_{_tableName}_GetBy_{indexName}", arParams, null, /* … MapReaderToList<T> */);

    // ExecuteAsync deriva parámetros desde la BD antes de asignarlos por posición:
    // SqlCommandBuilder.DeriveParameters(cmd);   // #L169
    // SetParameters(cmd, arParams);              // mapeo posicional, salta @RETURN_VALUE
}
```

**§13-B — Selección de proveedor + desencriptado de API key (sin secretos).** Fuente: `Providers/AIProviderFactory.cs#L17-L47`.

```csharp
public IAIProvider CreateProvider(Tenant tenant)
{
    var decryptedKey = DecryptApiKey(tenant.ApiKeyIA);
    return tenant.ProveedorIA.ToLower() switch
    {
        "gemini" => new GeminiProvider(decryptedKey, tenant.NombreModelo, tenant.Temperatura, tenant.MaxTokens),
        "claude" => new ClaudeProvider(_httpClientFactory.CreateClient("Claude"), decryptedKey, /* … */),
        "openai" => new OpenAIProvider(decryptedKey, tenant.NombreModelo, /* … */),
        _ => throw new ArgumentException($"Proveedor no soportado: {tenant.ProveedorIA}")
    };
}

private static string DecryptApiKey(string encryptedKey)
{
    var keyBase64 = Environment.GetEnvironmentVariable("IACONNECT_ENCRYPTION_KEY");
    if (string.IsNullOrEmpty(keyBase64)) return encryptedKey;   // ⚠ fallback: key en texto plano
    // AES-256-CBC / PKCS7; IV = primeros 16 bytes del ciphertext … // (recorte, sin material sensible)
}
```

---

## 7. Gap / observaciones

**Acoplamiento y patrón de datos**
- **Acoplamiento fuerte a SQL Server + stored procedures.** Todo pasa por `Microsoft.Data.SqlClient` y la convención `SP_{tabla}_{op}`. No hay ORM ni abstracción de proveedor de BD; migrar de motor implicaría reescribir `DataEntityCore` y todos los SPs.
- **`SqlCommandBuilder.DeriveParameters` en cada llamada** (`DataEntityCore.cs#L169`) hace un round-trip adicional a la BD para leer metadatos de parámetros del SP en cada operación (coste de rendimiento; no hay caché de metadatos).
- **Mapeo posicional frágil.** `SetParameters` (`#L195-L205`) depende del orden exacto entre `arParams` y la firma del SP; los elementos sobrantes (el `0` reservado) se ignoran en silencio. Un cambio de orden en el SP no da error de compilación.
- **Estado estático global.** El connection string es `static` (`#L16`); toda la app comparte una única configuración fijada al arranque. Correcto para single-tenant de BD, pero global mutable.
- **`csproj` con dependencia pendiente.** Hay un `PackageReference` comentado a un paquete `DataEntityCore` corporativo (`csproj#L13-L14`, "when corporate NuGet source is configured"): la clase actual es una copia in-repo provisional.
- **Inversión de capas** *(inferido)*: Infrastructure referencia `IAConnect.Application` (`csproj#L5`), lo inverso de la dependencia habitual.

**Proveedores IA**
- **`AIProviders:*` de `appsettings.json` no se usa** en Infrastructure (contradicción con lo esperado): el modelo/temperatura/tokens/API key vienen del registro `lut_Tenants`. Los `DefaultModel` (`gemini-2.5-flash`, `claude-3-sonnet-20240229`, `gpt-4`) están declarados pero no operan como defaults en el código. Riesgo: config engañosa.
- **`Encryption:AesKey` (config) vs `IACONNECT_ENCRYPTION_KEY` (env var):** el código solo lee la variable de entorno (`AIProviderFactory.cs#L35`); la clave de config queda muerta. Divergencia de configuración.
- **Detección de transitorios por texto en Gemini** (`GeminiProvider.cs#L212-L218`): buscar `"429"`/`"rate"` en el mensaje es frágil ante cambios del SDK. Claude usa códigos HTTP tipados (más robusto).
- **Timeout/HttpClient desparejo:** solo Claude usa `IHttpClientFactory` (cliente nombrado con timeout 60s). Gemini y OpenAI instancian sus propios clientes SDK por constructor de proveedor (potencial *socket exhaustion* / sin timeout uniforme). *(inferido)*
- **Solo Claude tiene retry propio con backoff HTTP;** OpenAI delega en el SDK. Comportamiento de resiliencia no homogéneo entre proveedores.

**Seguridad (sin exponer secretos)**
- **API keys de proveedor cifradas en BD, desencriptadas en runtime.** `ApiKey_IA` (columna de `lut_Tenants`) se descifra con **AES-256-CBC + PKCS7**, clave desde la env var `IACONNECT_ENCRYPTION_KEY` (Base64), con el **IV en los primeros 16 bytes** del ciphertext (`AIProviderFactory.cs#L33-L60`). No se copian claves ni endpoints con credenciales en este documento.
- **⚠️ Fallback a texto plano en ausencia de la clave.** Si `IACONNECT_ENCRYPTION_KEY` no está seteada, `DecryptApiKey` devuelve `encryptedKey` tal cual (`#L38-L39`), asumiendo que la key está en claro. Implicación: en entornos donde la env var no esté configurada, `ApiKey_IA` puede estar almacenada/tratada en **texto plano** en la BD. Debe garantizarse la presencia de la clave en todos los entornos no-dev.
- **Sin logging de secretos observado** en la pieza. Los mensajes de error de proveedor incluyen el *body* de error de la API remota (`ClaudeProvider.cs#L212-L214`): verificar que dichos bodies no arrastren datos sensibles al log de la capa superior *(inferido / fuera de esta pieza)*.
- **Modelo de amenaza de credenciales de BD:** el connection string vive en `ConnectionStrings:IAConnect`; su protección depende del *secret store* de la API (fuera de esta pieza).
