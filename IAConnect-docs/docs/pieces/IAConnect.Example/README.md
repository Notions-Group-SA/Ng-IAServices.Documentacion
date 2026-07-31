---
doc_id: PIECE-EXAMPLE-001
doc_type: piece-readme
title: "IAConnect.Example — Ejemplo de integración"
version: 1.0.0
status: draft
origin: reverse-engineered
confidence: high
owner: pendiente-asignacion
last_review: 2026-07-15
review_cycle_days: 180
audience: [dev, agentes-automaticos]
classification: uso-interno
traces: []
supersedes: null
---

# IAConnect.Example — Ejemplo de integración

## 1. Resumen ejecutivo

`IAConnect.Example` es una aplicación de **consola .NET 8** (`OutputType: Exe`,
`IAConnect.Example/IAConnect.Example.csproj#L4-L5`) que actúa como **cliente HTTP de demostración de
`IAConnect.API`**. No integra el widget (`IAConnect.ChatWidget`) ni un SDK tipado propio de la solución: usa
directamente `System.Net.Http.HttpClient` contra la URL base `http://localhost:5051`
(`IAConnect.Example/Program.cs#L6`), sin ninguna referencia de proyecto ni paquete NuGet adicional — el
`.csproj` no declara `ProjectReference` ni `PackageReference` alguno
(`IAConnect.Example/IAConnect.Example.csproj#L1-L10`).

Según el `docs-manifest.yaml` de la solución, la pieza está clasificada como `type: library` (comentada como
*sample / PoC de integración*) con criticidad `low`. En la práctica, lo que demuestra es **cómo consumir la
API HTTP de IAConnect** (health check, endpoint raíz, Swagger y navegación interactiva de endpoints), por lo
que su valor documental principal es servir de ejemplo de integración para consumidores de `IAConnect.API`,
no de una librería reutilizable en sí misma (no expone tipos ni métodos públicos para ser referenciada por
otro proyecto).

## 2. Qué hace paso a paso

Flujo completo tal como está en `IAConnect.Example/Program.cs` (top-level statements, `Main` implícito):

1. **Banner de inicio**: imprime un encabezado ASCII en consola (`Program.cs#L8-L13`).
2. **Crea el cliente HTTP**: `new HttpClient { BaseAddress = new Uri(API_BASE_URL) }` contra
   `http://localhost:5051` (`Program.cs#L15`).
3. **Health check**: llama a `TestHealthCheck(httpClient)`, que hace `GET /health` y reporta en color
   verde/rojo si la respuesta fue exitosa, o captura `HttpRequestException` si la API no está disponible
   (`Program.cs#L18`, función en `L39-L62`).
4. **Endpoint raíz**: llama a `TestRootEndpoint(httpClient)`, que hace `GET /` e imprime el cuerpo de la
   respuesta como texto (`Program.cs#L21`, función en `L64-L88`).
5. **Aviso de Swagger**: imprime en pantalla la URL de Swagger UI (`{API_BASE_URL}/swagger`) — solo texto
   informativo, no se invoca ese endpoint (`Program.cs#L23-L26`).
6. **Demo interactivo**: llama a `RunInteractiveDemo(httpClient)` (`Program.cs#L29`, función en `L90-L176`),
   que:
   - Define un arreglo fijo de 3 endpoints de ejemplo — `Health Check` (`GET /health`), `Root Info`
     (`GET /`) y `Swagger JSON` (`GET /swagger/v1/swagger.json`), todos con `Body: null`
     (`Program.cs#L96-L101`). El método también contempla el caso `POST` con cuerpo JSON serializado
     (`Program.cs#L126-L130`), aunque ningún endpoint de la lista actual lo usa.
   - Muestra un menú numerado y lee la opción del usuario por `Console.ReadLine()` en un bucle
     (`Program.cs#L103-L118`); `0` o entrada vacía termina el bucle.
   - Ejecuta la petición elegida, imprime el código de estado HTTP y, si el cuerpo de la respuesta es JSON
     válido, lo reformatea con sangría (`JsonSerializerOptions { WriteIndented = true }`); si no es JSON o
     supera 500 caracteres, lo trunca (`Program.cs#L142-L159`).
   - Cualquier excepción durante la llamada se captura y se imprime en rojo, sin interrumpir el bucle
     (`Program.cs#L162-L167`).
7. **Cierre**: imprime mensaje de finalización y espera una tecla (`Console.ReadKey()`) antes de salir
   (`Program.cs#L31-L34`).

## 3. Cómo ejecutarlo y configuración requerida

- **Requisito previo**: tener `IAConnect.API` corriendo localmente y escuchando en
  `http://localhost:5051` — esa URL está **hardcodeada** como constante `API_BASE_URL`
  (`Program.cs#L6`); no hay `appsettings.json`, variable de entorno ni argumento de línea de comandos que la
  parametrice. Para apuntar a otra URL/puerto hay que editar el código fuente.
- **Sin secretos ni credenciales**: el proyecto no lee cadenas de conexión, API keys ni tokens; todas las
  llamadas son anónimas (sin cabecera `Authorization`).
- **Ejecución**:
  ```
  dotnet run --project IAConnect.Example
  ```
  (o `F5`/`dotnet run` desde el propio directorio del proyecto). Es una app de consola interactiva: tras el
  health check y el endpoint raíz automáticos, queda esperando la selección de un número de menú por
  teclado (`stdin`).
- **Dependencias del proyecto**: ninguna (`IAConnect.Example.csproj#L1-L10` — sin `PackageReference` ni
  `ProjectReference`); solo usa BCL de .NET 8 (`System.Net.Http`, `System.Text.Json`, `System.Console`).
- **Target framework**: `net8.0`, `Nullable` y `ImplicitUsings` habilitados
  (`IAConnect.Example.csproj#L5-L7`).

## 4. Snippet de integración representativo (§13)

> Ejemplo — uso canónico de un *health check* HTTP contra `IAConnect.API`
> Fuente: `IAConnect.Example/Program.cs#L6, L15, L39-L62` @d49f677 · Demuestra: cómo instanciar un
> `HttpClient` apuntando a la API y sondear disponibilidad con manejo de errores de red

```csharp
const string API_BASE_URL = "http://localhost:5051";

using var httpClient = new HttpClient { BaseAddress = new Uri(API_BASE_URL) };

// ...

static async Task TestHealthCheck(HttpClient client)
{
    Console.Write("🏥 Health Check (/health) ... ");
    try
    {
        var response = await client.GetAsync("/health");
        if (response.IsSuccessStatusCode)
        {
            Console.ForegroundColor = ConsoleColor.Green;
            Console.WriteLine($"OK ({(int)response.StatusCode})");
        }
        else
        {
            Console.ForegroundColor = ConsoleColor.Red;
            Console.WriteLine($"FAIL ({(int)response.StatusCode})");
        }
    }
    catch (HttpRequestException ex)
    {
        Console.ForegroundColor = ConsoleColor.Red;
        Console.WriteLine($"ERROR: {ex.Message}");
    }
    Console.ResetColor();
}
```

**Qué demuestra**: el patrón mínimo para consumir `IAConnect.API` desde un cliente .NET — `HttpClient` con
`BaseAddress` fijo, una llamada `GetAsync` a una ruta relativa, verificación de `IsSuccessStatusCode` y
captura explícita de `HttpRequestException` para el caso en que la API no esté disponible (servidor caído,
DNS, conexión rechazada). Es el bloque más reutilizable de todo el ejemplo: **candidato natural para la guía
de implementación de la librería** (Marco §7.1 `library`, «ejemplos de integración extraídos de los
proyectos de prueba/PoC reales de la solución») cuando se documente `IAConnect.API` o un futuro SDK cliente.

**Precondición**: `IAConnect.API` corriendo y accesible en `http://localhost:5051` (o la URL que se
configure); no requiere autenticación para `/health`.

**Resultado esperado**: consola imprime `OK (200)` en verde si la API responde con código 2xx; `FAIL (<code>)`
en rojo si responde con otro código; `ERROR: <mensaje>` en rojo si la conexión falla (API no levantada,
puerto incorrecto, etc.). El mismo patrón (`try/GetAsync/IsSuccessStatusCode/catch HttpRequestException`) se
repite en `TestRootEndpoint` (`Program.cs#L64-L88`) y en el selector de endpoints del demo interactivo
(`Program.cs#L123-L167`), con el agregado de formateo *pretty-print* del JSON de respuesta
(`Program.cs#L145-L158`).

## 5. Observaciones / gap

- **URL de la API hardcodeada** (`Program.cs#L6`): no hay mecanismo de configuración externo
  (`appsettings.json`, variables de entorno, argumentos). Cualquier entorno distinto de
  `http://localhost:5051` exige editar y recompilar el código fuente. Gap para un ejemplo "productivo": se
  podría parametrizar vía `args[0]` o variable de entorno sin agregar dependencias.
- **`using` sin uso aparente**: `System.Net.Http.Json` (`Program.cs#L1`) y
  `System.Text.Json.Serialization` (`Program.cs#L4`) están importados pero no se detectó uso de sus
  extensiones (`GetFromJsonAsync`, `PostAsJsonAsync`, atributos de serialización) en el archivo — el consumo
  de JSON observado usa `System.Text.Json.JsonSerializer` directamente (`L128`, `L147-L149`). Posible
  remanente de una versión anterior del ejemplo; no afecta la compilación.
- **Rama `POST` no ejercitada**: el demo interactivo contempla el caso de endpoints `POST` con cuerpo
  (`Program.cs#L126-L130`), pero el arreglo de endpoints definidos (`L96-L101`) solo incluye `GET`. No hay
  evidencia en este archivo de un ejemplo de invocación a los endpoints de negocio de `IAConnect.API`
  (`/chat`, `/completion`, `/analyze`, `/summarize`, `/improve` según la descripción de la solución en
  `docs-manifest.yaml`) — el ejemplo cubre solo endpoints de infraestructura (health, root, swagger.json).
  Gap señalado para una futura ampliación del ejemplo o para el catálogo de API (`05-apis/catalog.md`).
- **Sin pruebas automatizadas ni *doc test***: no hay forma de verificar en CI que el ejemplo sigue
  compilando/corriendo contra la API real (Marco §13.4 punto 6 no se cumple); depende de ejecución manual.
- **Sin `CHANGELOG.md` en el origen**: consistente con `GAP-CHANGELOG` ya declarado en
  `docs-manifest.yaml` a nivel solución.
