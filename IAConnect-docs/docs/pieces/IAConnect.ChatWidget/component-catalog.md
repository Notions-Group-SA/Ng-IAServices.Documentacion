---
doc_id: PIECE-WIDGET-CAT-001
doc_type: component-catalog
title: IAConnect.ChatWidget — Catálogo de componentes
version: 1.0.0
status: draft
origin: reverse-engineered
confidence: high
owner: pendiente-asignacion
last_review: 2026-07-15
review_cycle_days: 180
audience: [dev, frontend, agentes-automaticos]
classification: uso-interno
traces: [PIECE-WIDGET-001, OVW-MAP-001]
supersedes: null
---

# IAConnect.ChatWidget — Catálogo de componentes

> Formato conforme al Marco de Documentación de Soluciones de Software §7.2 (entrada de catálogo de
> librería de componentes). Cada entrada lista únicamente los `[Parameter]` reales encontrados en el
> `.razor` correspondiente — no se agregan props inexistentes. `status: draft`, `origin:
> reverse-engineered` — trazable a `f:\repos\ng-sa\Workspace-GDA\NG\Ng-IAServices\IAConnect.ChatWidget`
> (origen no modificado).

## Índice

| Componente | Desde versión | Estado |
|---|---|---|
| [`<IAConnectChat />`](#iaconnectchat-) | no consta | estable (uso interno + público) |
| [`<IAConnectChatWidget />`](#iaconnectchatwidget-) | no consta | estable (público, punto de entrada recomendado) |

No hay `CHANGELOG.md` en el origen (gap `GAP-CHANGELOG`, ver `README.md#5`), por lo que **"desde versión"
no puede determinarse por historial**; el único dato de versión disponible es el `Version=1.0.1` del
paquete completo (`IAConnect.ChatWidget.csproj#L8`), no por componente. Se marca `no consta` en vez de
inferir.

---

## `<IAConnectChat />`

> Desde: no consta · Estado: estable · Fuente: `IAConnectChat.razor`
> A11y: botón de cierre con `aria-label`; envío por teclado (`Enter` sin `Shift`); **sin** `aria-label` en
> botón de envío ni `aria-live` en indicador de escritura/errores — ver `README.md#6` (GAP-A11Y)

Panel de chat completo: header configurable, historial de mensajes (renderizado Markdown de las
respuestas del asistente vía Markdig — `IAConnectChat.razor#L40`), input con textarea y botón de envío,
manejo de errores inline. Es el componente que consume la API de chat directamente
(`IIAConnectChatService`, `IIAConnectAuthService`, inyectados por DI — `IAConnectChat.razor#L128-L130`).
`IAConnectChatWidget` lo usa internamente como panel embebido (§`<IAConnectChatWidget />`).

### Parámetros

| Parámetro | Tipo | Default | Descripción | Fuente |
|---|---|---|---|---|
| `TenantId` | `string` | `""` (**`[EditorRequired]`**) | Identificador del tenant de IAConnect; se usa en las rutas `/api/ai/{tenantId}/chat` y `/api/tenants/{tenantId}`. | `IAConnectChat.razor#L92` |
| `ApiBaseUrl` | `string?` | Calculado: `GetDefaultUrl()` según `Environment` si no se asigna explícitamente | URL base de la API. Getter/setter custom: si no se seteó, deriva del `Environment` (`Sandbox`→desa, `Production`→prod). | `IAConnectChat.razor#L93-L98`, `#L296-L300` |
| `AccessToken` | `string?` | `null` | Token JWT directo ("modo legacy"). Si se provee, el componente lo usa sin autenticación interna. | `IAConnectChat.razor#L103` |
| `Credentials` | `IAConnectCredentials?` | `null` | Credenciales (`Username`/`Password`) para que el componente resuelva login/refresh internamente cuando `AccessToken` es `null`. | `IAConnectChat.razor#L109` |
| `Environment` | `IAConnectEnvironment` | `IAConnectEnvironment.Sandbox` | Entorno (`Sandbox`/`Production`); determina la URL por defecto cuando `ApiBaseUrl` no se asigna. | `IAConnectChat.razor#L112` |
| `Title` | `string` | `"Chat IA"` | Texto del header. | `IAConnectChat.razor#L115` |
| `HeaderColor` | `string` | `"#3b82f6"` | Color de fondo del header y del botón de envío (CSS inline). | `IAConnectChat.razor#L116` |
| `HeaderTextColor` | `string` | `"#ffffff"` | Color de texto del header y del botón de envío. | `IAConnectChat.razor#L117` |
| `UserLabel` | `string` | `"Tú"` | Etiqueta mostrada sobre los mensajes del usuario. | `IAConnectChat.razor#L118` |
| `AssistantLabel` | `string` | `"Asistente IA"` | Etiqueta mostrada sobre los mensajes del asistente. | `IAConnectChat.razor#L119` |
| `PlaceholderText` | `string` | `"Envíe un mensaje para comenzar"` | Texto mostrado en el área de mensajes cuando no hay historial. | `IAConnectChat.razor#L120` |
| `InputPlaceholder` | `string` | `"Escriba su mensaje..."` | Placeholder del `<textarea>` de entrada. | `IAConnectChat.razor#L121` |
| `SessionId` | `string?` | `null` (si es `null`, se genera `chat-{Guid:N}` en `OnInitializedAsync`) | ID de sesión de chat. | `IAConnectChat.razor#L122`, `#L152` |
| `ContainerStyle` | `string?` | `null` | CSS inline adicional aplicado al contenedor raíz (`style="@ContainerStyle"`). | `IAConnectChat.razor#L123`, `#L11` |
| `ShowCloseButton` | `bool` | `false` | Muestra/oculta el botón "✕" del header. | `IAConnectChat.razor#L124` |
| `OnClose` | `EventCallback` | — | Callback invocado al hacer clic en el botón de cierre. | `IAConnectChat.razor#L125`, `#L231-L234` |

### Miembros públicos adicionales

| Miembro | Tipo | Descripción | Fuente |
|---|---|---|---|
| `ResetSessionAsync()` | `Task` (método público) | Permite al host reiniciar la sesión: genera nuevo `SessionId`, limpia mensajes/errores y recarga el mensaje de bienvenida. Requiere referencia al componente (`@ref`) para invocarse desde el host. | `IAConnectChat.razor#L239-L246` |

### Ejemplo de uso

> Fuente: parámetros `[Parameter]` de `IAConnectChat.razor#L92-L125` (proyección, no transclusión —
> Marco §13.1 mecanismo 2, copia verificada con procedencia)

```razor
<IAConnectChat TenantId="tenant-demo"
               AccessToken="@jwtDelHost"
               Title="Chat IA"
               ShowCloseButton="true"
               OnClose="() => panelAbierto = false" />
```

### Estados observables

- **Cargando inicial** (`_loading = true` hasta que `OnInitializedAsync` resuelve auth + mensaje de
  bienvenida — `IAConnectChat.razor#L150-L156`): mientras `_loading` es `true` y no hay mensajes, se
  muestra el `PlaceholderText`.
- **Vacío**: sin mensajes y no cargando → bloque `.iaconnect-chat-empty` con `PlaceholderText`
  (`IAConnectChat.razor#L23-L28`).
- **Enviando** (`_sending = true`): textarea y botón deshabilitados; el botón muestra un spinner en vez
  del glifo de envío; aparece una burbuja de "escribiendo" con tres puntos animados
  (`IAConnectChat.razor#L50-L57`, `#L63-L79`).
- **Error**: `_error` no nulo → línea de texto `Error: {mensaje}` o `Error de autenticación: {mensaje}`
  bajo el input (`IAConnectChat.razor#L81-L84`, `#L213-L216`, `#L292`). Es texto plano, no asociado por
  `aria-describedby` al input (gap A11y, ver `README.md#6`).
- **Deshabilitado por vacío**: el botón de envío también se deshabilita si el textarea está vacío/en
  blanco (`IAConnectChat.razor#L69`), independientemente de `_sending`.

---

## `<IAConnectChatWidget />`

> Desde: no consta · Estado: estable · Fuente: `IAConnectChatWidget.razor`
> A11y: el avatar/burbuja flotante es un `<div @onclick>` sin `role="button"` ni soporte de teclado — ver
> `README.md#6` (GAP-A11Y). Imagen de avatar con `alt` cuando se usa `AvatarImageUrl`; el SVG por defecto
> no tiene nombre accesible.

Widget flotante de tipo "burbuja + ventana": muestra un avatar circular fijo (posición configurable por
CSS) que, al hacer clic, abre una ventana con `IAConnectChat` embebido (`ShowCloseButton="true"` fijo,
`IAConnectChatWidget.razor#L52`). Es el componente pensado como punto de entrada único para incrustar en
una página host (`OVW-MAP-001`: relación `Demo.Web → ChatWidget → IAConnect.API`).

### Parámetros

| Parámetro | Tipo | Default | Descripción | Fuente |
|---|---|---|---|---|
| `TenantId` | `string` | `""` (**`[EditorRequired]`**) | Igual que en `IAConnectChat`; se pasa tal cual al panel embebido. | `IAConnectChatWidget.razor#L62`, `#L38` |
| `ApiBaseUrl` | `string?` | Calculado: `GetDefaultUrl()` según `Environment` si no se asigna | Igual semántica que en `IAConnectChat` (getter/setter custom duplicado). | `IAConnectChatWidget.razor#L63-L68`, `#L105-L109` |
| `AccessToken` | `string?` | `null` | Pasado tal cual a `IAConnectChat`. | `IAConnectChatWidget.razor#L69`, `#L40` |
| `Credentials` | `IAConnectCredentials?` | `null` | Pasado tal cual a `IAConnectChat`. | `IAConnectChatWidget.razor#L70`, `#L41` |
| `Environment` | `IAConnectEnvironment` | `IAConnectEnvironment.Sandbox` | Igual semántica que en `IAConnectChat`. | `IAConnectChatWidget.razor#L73`, `#L42` |
| `WindowWidth` | `int` | `380` | Ancho en px de la ventana de chat (variable CSS `--iaconnect-window-w`). | `IAConnectChatWidget.razor#L76`, `#L37` |
| `WindowHeight` | `int` | `520` | Alto en px de la ventana de chat (variable CSS `--iaconnect-window-h`). | `IAConnectChatWidget.razor#L77`, `#L37` |
| `AvatarSize` | `int` | `52` | Tamaño en px del avatar circular (variable CSS `--iaconnect-avatar-size`). | `IAConnectChatWidget.razor#L78`, `#L8` |
| `AvatarImageUrl` | `string?` | `null` (si es `null`/vacío, se renderiza un SVG de robot inline por defecto) | URL de imagen custom para el avatar. | `IAConnectChatWidget.razor#L79`, `#L9-L30` |
| `PositionBottom` | `string` | `"24px"` | Offset `bottom` (CSS) del wrapper posicionado `fixed`. | `IAConnectChatWidget.razor#L82`, `#L99-L100` |
| `PositionRight` | `string` | `"24px"` | Offset `right` (CSS) del wrapper posicionado `fixed`. | `IAConnectChatWidget.razor#L83`, `#L99-L100` |
| `Title` | `string` | `"Chat IA"` | Pasado tal cual a `IAConnectChat`. | `IAConnectChatWidget.razor#L86`, `#L43` |
| `HeaderColor` | `string` | `"#3b82f6"` | Pasado tal cual a `IAConnectChat`. | `IAConnectChatWidget.razor#L87`, `#L44` |
| `HeaderTextColor` | `string` | `"#ffffff"` | Pasado tal cual a `IAConnectChat`. | `IAConnectChatWidget.razor#L88`, `#L45` |
| `UserLabel` | `string` | `"Tú"` | Pasado tal cual a `IAConnectChat`. | `IAConnectChatWidget.razor#L89`, `#L46` |
| `AssistantLabel` | `string` | `"Asistente IA"` | Pasado tal cual a `IAConnectChat`. | `IAConnectChatWidget.razor#L90`, `#L47` |
| `PlaceholderText` | `string` | `"Envíe un mensaje para comenzar"` | Pasado tal cual a `IAConnectChat`. | `IAConnectChatWidget.razor#L91`, `#L48` |
| `InputPlaceholder` | `string` | `"Escriba su mensaje..."` | Pasado tal cual a `IAConnectChat`. | `IAConnectChatWidget.razor#L92`, `#L49` |
| `SessionId` | `string?` | `null` | Pasado tal cual a `IAConnectChat`. | `IAConnectChatWidget.razor#L93`, `#L50` |
| `ContainerStyle` | `string?` | `null` | Pasado tal cual a `IAConnectChat` (estilo del panel interno, no del wrapper flotante). | `IAConnectChatWidget.razor#L94`, `#L51` |

**Nota:** este componente **no expone** `ShowCloseButton` ni `OnClose` como parámetros propios — internamente
siempre pasa `ShowCloseButton="true"` a `IAConnectChat` y conecta `OnClose` a su propio método `Close()`
(cierra la ventana y vuelve a mostrar el avatar) — `IAConnectChatWidget.razor#L52-L53`, `#L103`.

### Ejemplo de uso

> Fuente: parámetros `[Parameter]` de `IAConnectChatWidget.razor#L62-L94`

```razor
<IAConnectChatWidget TenantId="tenant-demo"
                      Environment="IAConnectEnvironment.Sandbox"
                      Credentials="new IAConnectCredentials { Username = user, Password = pass }"
                      Title="Asistente Virtual"
                      HeaderColor="#1a5276" />
```

### Estados observables

- **Cerrado** (`_open = false`, estado inicial): se muestra solo el avatar circular
  (`IAConnectChatWidget.razor#L6-L32`); animación CSS continua `iaconnect-avatarBounce` de "llamado de
  atención" (`IAConnectChatWidget.razor.css#L20`, `#L33-L36`).
- **Abierto** (`_open = true`, toggle por clic en el avatar — `IAConnectChatWidget.razor#L8`, `#L102`): se
  oculta el avatar y se muestra la ventana con `IAConnectChat` embebido, con animación de entrada
  `iaconnect-slideUp` (`IAConnectChatWidget.razor.css#L73`, `#L77-L80`).
- **Responsive / mobile** (`max-width: 480px`): la ventana pasa a ocupar toda la pantalla
  (`position: fixed`, `100vw`/`100dvh`, sin animación) y el avatar se oculta mientras la ventana está
  presente en el árbol — comportamiento del CSS con *scope* (`IAConnectChatWidget.razor.css#L90-L110`); no
  hay lógica C# adicional para este breakpoint, es puramente CSS.
- **Avatar por defecto vs. custom**: si `AvatarImageUrl` es nulo/vacío, se renderiza un `<svg>` inline
  (robot); si tiene valor, se renderiza `<img src="@AvatarImageUrl" alt="Asistente Virtual" />`
  (`IAConnectChatWidget.razor#L9-L30`). El asset `wwwroot/images/asistente-virtual-trabajo.jpg` existe en
  el proyecto pero **no se encontró referencia a él en ningún `.razor`** — `[inferido]`: parece un asset
  de referencia/diseño no conectado por defecto; el host debería pasarlo explícitamente vía
  `AvatarImageUrl` si quiere usarlo, marcado como observación sin confirmar intención.
