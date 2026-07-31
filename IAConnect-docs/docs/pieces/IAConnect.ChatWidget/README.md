---
doc_id: PIECE-WIDGET-001
doc_type: piece-readme
title: IAConnect.ChatWidget — Widget de chat Blazor
version: 1.0.0
status: draft
origin: reverse-engineered
confidence: high
owner: pendiente-asignacion
last_review: 2026-07-15
review_cycle_days: 180
audience: [dev, frontend, agentes-automaticos]
classification: uso-interno
traces: [OVW-MAP-001, GAP-CHANGELOG, GAP-A11Y]
supersedes: null
---

# IAConnect.ChatWidget — Widget de chat Blazor

> **Estado documental:** `draft`, reconstruido por ingeniería inversa desde el código fuente
> (`/NG/Ng-IAServices/IAConnect.ChatWidget`, **no modificado**). Todo lo marcado `[inferido]` es una
> lectura razonada del código, no un hecho declarado explícitamente en él; requiere validación humana.

## 1. Resumen ejecutivo

**IAConnect.ChatWidget** es una **Razor Class Library (RCL)** para .NET 8 que empaqueta un widget de chat
embebible en aplicaciones Blazor. Se distribuye como paquete NuGet `Fito.ChatWidget` v1.0.1
(`IAConnect.ChatWidget.csproj#L7-L8`), descripto en el propio `.csproj` como el "componente Blazor de chat
con IA para **Fito** de Notions Group S.A." que "se conecta directamente a la API REST sin necesidad de
implementar interfaces" (`IAConnect.ChatWidget.csproj#L10`). Es decir: aunque el namespace y la solución son
`IAConnect`, el paquete publicado está orientado a un producto/cliente llamado **Fito** — dato trazable al
`.csproj`, no inferido.

Expone dos componentes Razor públicos (catálogo completo en
[`component-catalog.md`](./component-catalog.md)):

- **`IAConnectChat`** (`IAConnectChat.razor`): el panel de chat completo (header, mensajes, input).
- **`IAConnectChatWidget`** (`IAConnectChatWidget.razor`): un contenedor flotante (avatar/burbuja +
  ventana) que envuelve a `IAConnectChat` y agrega abrir/cerrar (`IAConnectChatWidget.razor#L6-L55`).

**Cómo se embebe:** el host (una app Blazor Server o WASM) registra los servicios del widget vía
`AddIAConnectChatWidget(...)` en el contenedor de DI y luego coloca `<IAConnectChatWidget />` o
`<IAConnectChat />` en cualquier página/layout Razor (§2).

**Qué API consume:** el widget habla HTTP directo contra la API REST de IAConnect (ver
`IAConnect.API`, pieza `service` — `OVW-MAP-001`). No depende de un cliente SDK externo: implementa su
propio `HttpClient` vía `IHttpClientFactory` (`Services/IAConnectHttpChatService.cs`,
`Services/IAConnectHttpAuthService.cs`). Endpoints observados:

| Endpoint | Método | Uso | Fuente |
|---|---|---|---|
| `/api/auth/login` | POST | Login con credenciales → tokens | `Services/IAConnectHttpAuthService.cs#L29` |
| `/api/auth/refresh` | POST | Renovación de access token | `Services/IAConnectHttpAuthService.cs#L66` |
| `/api/ai/{tenantId}/chat` | POST | Envío de mensaje de chat | `Services/IAConnectHttpChatService.cs#L25` |
| `/api/tenants/{tenantId}` | GET | Config del tenant (mensaje de bienvenida) | `Services/IAConnectHttpChatService.cs#L35` |

La URL base por defecto, si no se configura explícitamente, depende del `Environment` del componente
(`Sandbox` → `https://desa-fito.notionsgroup.com.ar`, `Production` → `https://fito.notionsgroup.com.ar`):
`IAConnectChat.razor#L296-L300`, replicado en `IAConnectChatWidget.razor#L105-L109`. Estas URLs están
*hardcodeadas* en el código de ambos componentes (duplicación literal, no un solo punto de verdad) —
señalado como observación, no como recomendación de cambio.

## 2. Instalación y uso

### 2.1 Registro en el contenedor de DI

La API pública de registro vive en `Extensions/ServiceCollectionExtensions.cs` (namespace
`IAConnect.ChatWidget.Extensions`, clase `IAConnectChatWidgetExtensions`). Expone dos sobrecargas del
método de extensión **`AddIAConnectChatWidget`**:

| Firma | Descripción | Fuente |
|---|---|---|
| `AddIAConnectChatWidget(this IServiceCollection services)` | Registra con opciones por defecto (delega en la sobrecarga con `configure`) | `Extensions/ServiceCollectionExtensions.cs#L16-L19` |
| `AddIAConnectChatWidget(this IServiceCollection services, Action<IAConnectChatWidgetOptions> configure)` | Registra permitiendo configurar `IAConnectChatWidgetOptions` | `Extensions/ServiceCollectionExtensions.cs#L35-L44` |

Internamente, ambas sobrecargas terminan registrando (`Extensions/ServiceCollectionExtensions.cs#L39-L43`):

- `services.Configure(configure)` → *binding* de `IAConnectChatWidgetOptions` a `IOptions<T>`.
- `services.AddHttpClient()` → habilita `IHttpClientFactory` (los servicios HTTP internos lo consumen
  con el nombre lógico `"IAConnectChatWidget"`; ver §4).
- `services.AddScoped<IIAConnectChatService, IAConnectHttpChatService>()`.
- `services.AddScoped<IIAConnectAuthService, IAConnectHttpAuthService>()`.

> Ejemplo de uso — registro con configuración
> Fuente: `Extensions/ServiceCollectionExtensions.cs#L26-L34` (docstring `<example>` del propio código,
> copiado literal como comentario XML documental, no como código ejecutado por el proyecto)

```csharp
builder.Services.AddIAConnectChatWidget(options =>
{
    options.ApiBaseUrl = "https://api.miempresa.com";
    options.CustomCssUrl = "https://cdn.miempresa.com/estilos/chat.css";
});
```

Como ambos servicios (`IIAConnectChatService`, `IIAConnectAuthService`) son interfaces públicas con
implementación HTTP por defecto, un consumidor puede reemplazarlas registrando su propia implementación
**después** de `AddIAConnectChatWidget(...)` (el último registro de `AddScoped` gana) — así lo documenta
el propio código: *"Los consumidores pueden registrar su propia implementación si necesitan comportamiento
personalizado"* (`Services/IIAConnectChatService.cs#L8`). `[inferido]`: el mecanismo de reemplazo (registrar
después) no está probado en el origen, se deduce del comportamiento estándar de `IServiceCollection`.

### 2.2 Colocación del componente

Los `@using` que el consumidor necesita ya están resueltos dentro de la librería vía `_Imports.razor`
(`_Imports.razor#L1-L7`: `Microsoft.AspNetCore.Components.Web`, `Microsoft.AspNetCore.Components.Forms`,
`Microsoft.Extensions.Options`, `Markdig`, y los namespaces propios `Configuration`, `Services`,
`Models`). El host solo necesita `@using IAConnect.ChatWidget` (o el `using` correspondiente) para el tag
del componente en sí.

> Ejemplo de uso — widget flotante embebido en una página/layout
> Fuente: parámetros `[Parameter]` de `IAConnectChatWidget.razor#L62-L94`

```razor
<IAConnectChatWidget TenantId="tenant-demo"
                      Environment="IAConnectEnvironment.Sandbox"
                      Credentials="new IAConnectCredentials { Username = user, Password = pass }"
                      Title="Asistente Virtual"
                      HeaderColor="#1a5276" />
```

> Ejemplo de uso — panel de chat embebido directamente (sin burbuja flotante), con token ya resuelto
> Fuente: parámetros `[Parameter]` de `IAConnectChat.razor#L92-L125`

```razor
<IAConnectChat TenantId="tenant-demo"
               AccessToken="@jwtDelHost"
               Title="Chat IA"
               ShowCloseButton="true"
               OnClose="() => panelAbierto = false" />
```

Ambos ejemplos son *proyecciones* del código fuente (Marco §13): se listan los parámetros reales
encontrados en cada `.razor`, sin agregar props inexistentes. Catálogo completo de parámetros por
componente → [`component-catalog.md`](./component-catalog.md).

## 3. Opciones de configuración

Fuente: `Configuration/IAConnectChatWidgetOptions.cs#L7-L23`. Se configuran mediante el delegado
`configure` de `AddIAConnectChatWidget(...)` (§2.1) y se inyectan en los componentes como
`IOptions<IAConnectChatWidgetOptions>` (`IAConnectChat.razor#L130`).

| Propiedad | Tipo | Default | Descripción | Fuente |
|---|---|---|---|---|
| `ApiBaseUrl` | `string?` | `null` | URL base de la API de IAConnect. Si se configura, es el valor por defecto cuando el componente no recibe el parámetro `ApiBaseUrl` explícito. | `Configuration/IAConnectChatWidgetOptions.cs#L14` |
| `CustomCssUrl` | `string?` | `null` | URL de un CSS externo (absoluta o relativa al host) que se carga con prioridad sobre los estilos embebidos del componente. | `Configuration/IAConnectChatWidgetOptions.cs#L22` |

**Observación (gap):** `IAConnectChatWidgetOptions.ApiBaseUrl` está declarado en el modelo de opciones y
documentado como "valor por defecto cuando el componente no recibe `ApiBaseUrl`" (comentario XML,
`Configuration/IAConnectChatWidgetOptions.cs#L11-L13`), pero **no se encontró código en
`IAConnectChat.razor` ni `IAConnectChatWidget.razor` que lea `WidgetOptions.Value.ApiBaseUrl`** — el
getter de `ApiBaseUrl` en ambos componentes solo hace *fallback* a `GetDefaultUrl()` (URLs hardcodeadas por
`Environment`), no a la opción configurada (`IAConnectChat.razor#L93-L98`, `#L296-L300`;
`IAConnectChatWidget.razor#L63-L68`, `#L105-L109`). Se documenta como **divergencia observada entre el
comentario XML y el comportamiento del código**, no como hecho verificado en ambos sentidos; requiere
confirmación humana antes de asumir cuál de los dos es el comportamiento intencional.

`WidgetOptions.Value.CustomCssUrl` sí se usa: si no es nulo/vacío, `IAConnectChat.razor` inyecta un
`<link rel="stylesheet">` con esa URL antes del marcado del widget (`IAConnectChat.razor#L6-L9`).

## 4. Servicios internos (Auth / Chat HTTP)

El paquete registra dos servicios `Scoped` con implementación HTTP por defecto, ambos detrás de interfaz
pública para permitir reemplazo por el consumidor (§2.1):

| Interfaz | Implementación por defecto | Responsabilidad | Fuente |
|---|---|---|---|
| `IIAConnectChatService` | `IAConnectHttpChatService` | Enviar mensajes de chat y obtener config del tenant | `Services/IIAConnectChatService.cs`, `Services/IAConnectHttpChatService.cs` |
| `IIAConnectAuthService` | `IAConnectHttpAuthService` | Login y refresh de tokens JWT | `Services/IIAConnectAuthService.cs`, `Services/IAConnectHttpAuthService.cs` |

Ambas implementaciones crean su `HttpClient` vía `IHttpClientFactory.CreateClient("IAConnectChatWidget")`
(`Services/IAConnectHttpChatService.cs#L42`, `Services/IAConnectHttpAuthService.cs#L98`) — nombre lógico
compartido, sin configuración adicional de *Polly*/reintentos observada en el origen. Timeout: 120s para
el envío de mensajes de chat (`Services/IAConnectHttpChatService.cs#L46`), 30s para login/refresh
(`Services/IAConnectHttpAuthService.cs#L100`).

### 4.1 Cómo autentica el widget contra la API (sin secretos)

`IAConnectChat` soporta **dos modos de autenticación**, resueltos en `EnsureAuthenticatedAsync()`
(`IAConnectChat.razor#L256-L294`):

1. **Token directo (modo legacy):** si el host pasa el parámetro `AccessToken`, el componente lo usa tal
   cual y **no** intenta login ni refresh (`IAConnectChat.razor#L103`, `#L258-L260`). El host es
   responsable de obtener y renovar ese JWT por su cuenta.
2. **Credenciales resueltas internamente:** si se pasa `Credentials` (`Username`/`Password`,
   `Models/IAConnectCredentials.cs#L7-L14`) y no hay `AccessToken`, el componente:
   - reutiliza el token en memoria si no expiró (margen de 60s antes del vencimiento —
     `IAConnectChat.razor#L266-L267`);
   - si no, intenta `AuthService.RefreshAsync(...)` con el `refreshToken` en memoria
     (`IAConnectChat.razor#L270-L280`);
   - si no hay refresh token o falla, hace `AuthService.LoginAsync(...)` con las credenciales
     (`IAConnectChat.razor#L283-L293`).

El login (`POST /api/auth/login`) envía el body con las claves JSON `nombreUsuario` / `contraseña`
(`Models/AuthModels.cs#L9-L16`, vía `[JsonPropertyName]`); la respuesta trae `accessToken`,
`refreshToken`, `expiresIn` y `refreshExpiresIn` (`Models/AuthModels.cs#L30-L43`). El access token se
envía en llamadas de chat como header `Authorization: Bearer {token}`
(`Services/IAConnectHttpChatService.cs#L44-L45`).

**Nota de seguridad (sin secretos en esta doc):** el token y el `refreshToken` se mantienen únicamente en
memoria del componente (campos privados `_resolvedToken` / `_refreshToken`,
`IAConnectChat.razor#L142-L144`); no se observó persistencia en `localStorage`/cookies dentro de este
proyecto. Las credenciales (`Username`/`Password`) las provee el host como parámetro de componente — de
dónde las obtiene el host (formulario, secret manager, etc.) es responsabilidad fuera del alcance de esta
pieza y no se documenta aquí ningún valor real.

## 5. Política de breaking changes / SemVer

- El `.csproj` fija `PackageId=Fito.ChatWidget`, `Version=1.0.1` (`IAConnect.ChatWidget.csproj#L7-L8`) —
  versionado SemVer de hecho (formato `MAYOR.MENOR.PARCHE`), pero **no se encontró en el origen una
  política de breaking changes escrita** (ni criterios de qué constituye MAYOR/MENOR/PARCHE para este
  paquete, ni matriz de compatibilidad, ni guía de migración entre versiones).
- **No existe `CHANGELOG.md`** en `IAConnect.ChatWidget/` — confirmado por listado de archivos del
  proyecto (§ver árbol de fuente). Esto está declarado como gap global de la solución en
  `docs-manifest.yaml` (`known_gaps: GAP-CHANGELOG` — "Las piezas no tienen CHANGELOG.md en el origen; no
  se reconstruye historial de versiones"). El manifiesto de esta pieza lo confirma explícitamente:
  `pieces: IAConnect.ChatWidget.required_docs` incluye `changelog`, y `gap` lo lista como pendiente
  (`docs-manifest.yaml#L59-L61`).
- Según el perfil `component-library` del Marco de Documentación (§7.1), esta pieza debería regirse por
  **SemVer estricto** (los consumidores dependen del contrato de props/parámetros) y llevar Keep a
  Changelog — ambos son **recomendación del marco documental, no un hecho verificado en el código**.
- **GAP declarado:** no hay evidencia en el origen de una política formal de deprecación de `[Parameter]`
  ni de convivencia de versiones para los consumidores del paquete NuGet. Requiere decisión humana antes
  de publicar una versión 2.x.

## 6. Accesibilidad

Lo siguiente es **verificable directamente en el marcado Razor** (no es una auditoría de accesibilidad
formal):

| Aspecto | Observado | Fuente |
|---|---|---|
| Botón de cierre con etiqueta | `<button class="iaconnect-chat-close" @onclick="OnCloseClicked" aria-label="Cerrar">` — tiene `aria-label` | `IAConnectChat.razor#L17` |
| Envío por teclado | `@onkeydown="OnKeyDown"` en el `<textarea>`: `Enter` (sin `Shift`) envía el mensaje | `IAConnectChat.razor#L64`, `#L223-L229` |
| Placeholder de input | `<textarea placeholder="@InputPlaceholder" ...>` | `IAConnectChat.razor#L63-L67` |
| Estado deshabilitado visible | `disabled="@_sending"` en textarea y botón de envío durante el envío | `IAConnectChat.razor#L66`, `#L69` |
| Imagen de avatar con alt | `<img src="@AvatarImageUrl" alt="Asistente Virtual" />` (solo si `AvatarImageUrl` está seteado) | `IAConnectChatWidget.razor#L11` |
| Avatar SVG por defecto | El SVG robot inline (`IAConnectChatWidget.razor#L16-L29`) **no tiene** `role="img"` ni `aria-label`/`<title>` — no es anunciado por lectores de pantalla con un nombre accesible | `IAConnectChatWidget.razor#L16-L29` |
| Toggle de avatar/burbuja | El `<div class="iaconnect-widget-avatar" @onclick="Toggle">` (`IAConnectChatWidget.razor#L8`) es un `<div>` clickeable, **sin** `role="button"`, sin `tabindex`, sin manejador `@onkeydown` — no es operable por teclado ni anunciado como control interactivo | `IAConnectChatWidget.razor#L8-L31` |
| Botón de envío sin texto accesible | El botón de envío usa un glifo `➤` / spinner como único contenido, **sin** `aria-label` (a diferencia del botón de cierre) | `IAConnectChat.razor#L68-L79` |
| Indicador de "escribiendo" | Los `iaconnect-dot` (`IAConnectChat.razor#L52-L56`) son puramente visuales, sin texto ni `aria-live` que anuncie el estado a lectores de pantalla | `IAConnectChat.razor#L50-L57` |
| Idioma de la interfaz | Textos por defecto en español (`Title`, labels, placeholders) hardcodeados como valores de `[Parameter]`, configurables por el host pero sin mecanismo de `lang`/i18n observado | `IAConnectChat.razor#L115-L121` |

**GAP-A11Y (declarado en `docs-manifest.yaml`):** *"Declaración de accesibilidad (WCAG) de Widget y
Demo.Web pendiente de auditoría de accesibilidad real."* Esta sección **no reemplaza** esa auditoría:
documenta únicamente lo verificable por lectura de código (roles ARIA presentes/ausentes, soporte de
teclado observado). Pendiente explícito:

- Auditoría WCAG 2.x nivel AA formal (herramienta automatizada + revisión manual con lector de pantalla).
- Definir si el `<div>` avatar/burbuja (`IAConnectChatWidget.razor#L8`) debe convertirse en `<button>` o
  recibir `role="button"` + `tabindex="0"` + manejo de `Enter`/`Space`.
- `aria-label` para el botón de envío y `aria-live="polite"` para el indicador de "escribiendo" y para
  mensajes de error (`_error`, `IAConnectChat.razor#L81-L84`).
- Contraste de color no evaluado (colores configurables vía `HeaderColor`/`HeaderTextColor`, por lo que el
  contraste depende de la configuración del host, no solo del componente).
