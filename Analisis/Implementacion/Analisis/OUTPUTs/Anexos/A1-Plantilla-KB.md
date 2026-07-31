---
doc_id: ANX-A1
doc_type: template
title: Plantilla genérica comentada para fichas de base de conocimiento
status: draft
origin: ai-generated
confidence: high
owner: Administrador funcional de KB
last_review: 2026-07-31
audience: [administrador-funcional-kb, referente-funcional]
traces:
  - ../03-Estructura-y-Plantilla-KB.md
  - ../../../GDA-Turnos/06-Administrator-Guide.md
---

# A1 · Plantilla genérica de ficha de KB

Plantilla independiente del dominio: sirve para turnos, multas, habilitaciones o cualquier catálogo de trámites. Cada campo va acompañado de la pregunta que hay que responder para completarlo bien y de la razón por la que existe. El fundamento de cada regla está en [`03-Estructura-y-Plantilla-KB.md`](../03-Estructura-y-Plantilla-KB.md).

**Restricción de partida:** una ficha = un tema = **300–350 palabras**. Si no entra, son dos fichas.

---

## 1. Plantilla base

```markdown
<!--
kb_id: <TIPO>-<NNN>          # p. ej. T1-012
tipo: T1|T2|T3|T4|T5|T6|T7|T8
perfil: ciudadano | funcionario | ambos
fuente: <tabla/archivo/código del que se derivó, o "curado">
dueño: <nombre del responsable funcional>
vigencia: <fecha de última verificación contra la fuente>
-->

## <Título: la pregunta tal como la haría el usuario, o el nombre del ítem>

<Respuesta directa en una o dos frases. Si el usuario solo lee esto, ya resolvió.>

**También conocido como:** <palabras coloquiales, con y sin acento, singular y
plural, incluidos los errores de escritura frecuentes>

**Nombre exacto en el sistema:** <literal, tal como figura en la base>

**Detalle:**
<Desarrollo. Pasos numerados si es un procedimiento; condiciones si es una
regla; datos si es una ficha de catálogo. Un concepto por párrafo.>

**Enlace:** <URL completa, con el casing exacto que emite el código>
<Una línea diciendo qué va a encontrar ahí.>

**Lo que NO se puede:** <límites reales del sistema relacionados con este tema>

**Ojo:** <la advertencia que un humano con experiencia daría antes de que el
usuario actúe>
```

---

## 2. Preguntas que guían cada campo

| Campo | Pregunta a responder | Si no la podés responder… |
|---|---|---|
| `kb_id` | ¿Este contenido ya existe en otra ficha? | Estás por duplicar: buscá primero |
| `tipo` | ¿Es catálogo, procedimiento, regla, error o límite? | Si es más de uno, son varias fichas |
| `perfil` | ¿A quién le sirve esta respuesta tal como está redactada? | Si sirve a los dos con la misma voz, revisá: rara vez es cierto |
| `fuente` | ¿De dónde salió este dato y cómo lo verifico de nuevo? | Sin fuente no hay forma de saber cuándo dejó de ser cierto |
| `dueño` | ¿Quién decide si este texto es correcto? | Un contenido sin dueño está vencido y nadie lo sabe |
| `vigencia` | ¿Cuándo se verificó contra la fuente por última vez? | Sin fecha no se puede priorizar la revisión |
| **Título** | ¿Cómo escribiría esto un usuario que no conoce el sistema? | Si usás el nombre interno, la ficha no se recupera |
| **Respuesta directa** | Si el usuario lee solo la primera línea, ¿le alcanza? | El chat corta la atención antes de lo que suponés |
| **También conocido como** | ¿Qué cinco palabras distintas usaría la gente para esto? | Con RAG léxico, sin la palabra no hay recuperación |
| **Nombre exacto** | ¿Coincide carácter por carácter con la base, incluidas las tildes que faltan? | El usuario no va a encontrar en pantalla lo que el asistente le nombró distinto |
| **Detalle** | ¿Se entiende leído aislado, sin la ficha anterior? | El fragmento viaja solo: no hay «como se explicó antes» |
| **Enlace** | ¿Es la ruta del canal donde está montado el widget? | Un enlace del portal en la app lleva a un 404 |
| **Lo que NO se puede** | ¿Qué va a pedir el usuario en este tema que el sistema no hace? | El silencio se llena con alucinación |
| **Ojo** | ¿Qué le diría un empleado con experiencia antes de que apriete el botón? | Es el campo que más consultas evita y el que casi nadie escribe |

---

## 3. Variantes por tipo

Los campos obligatorios cambian según el tipo de ficha.

| Tipo | Campos obligatorios adicionales | Campos que sobran |
|---|---|---|
| **T1 · Catálogo** | Nombre exacto, alias, dónde se atiende, enlace | Pasos |
| **T2 · Sinónimos** | Tabla `término del usuario → nombre del sistema → id` | Detalle narrativo |
| **T3 · Requisitos** | Lista de requisitos, texto des-HTMLizado de la fuente | Alias (los toma de T1) |
| **T4 · Procedimiento** | Pasos numerados, precondición, resultado esperado | — |
| **T5 · Regla de negocio** | Condición, consecuencia, dónde se parametriza | Enlace, si no hay pantalla asociada |
| **T6 · Mensaje de error** | **El mensaje literal como título**, causa, qué hacer | Alias |
| **T7 · Límite** | Verbos coloquiales del concepto negado, alternativa real | Pasos |
| **T8 · Rutas** | Tabla `pantalla → ruta → canal` | Narrativa |

### 3.1 T2 — Sinónimos, formato recomendado

```markdown
## Cómo se dice cada trámite

| Como lo dice la gente | Nombre en el sistema | Id |
|---|---|---|
| registro, carnet, carnet de manejar, licencia de manejar | Licencia de Conducir | 12 |
| castrar, capar, esterilizar, castracion, castración | <Nombre exacto> | <Id> |
| veterinario, veterinaria, zoonosis, sanidad animal | <Nombre exacto> | <Id> |
```

🟨 Esta ficha es el activo más reusable de todo el corpus, y el más frágil: si alguien renombra o da de baja un ítem en el ABM, la tabla queda mintiendo en silencio. Necesita un job de verificación contra la tabla de origen.

### 3.2 T6 — Mensajes de error, formato recomendado

El **título es el mensaje literal**, entre comillas, tal como el usuario lo vio en pantalla. Es lo que va a pegar en el chat, y es lo que hace que el fragmento se recupere.

```markdown
## "Otro usuario esta reservando este turno. Volvé mas tarde o elegí otro."

<Qué está pasando, en lenguaje del usuario.>

**Qué hacer:** <alternativas concretas.>
```

⚠️ Copiar el mensaje **con sus errores de ortografía si los tiene**. La ficha no es el lugar para corregir al sistema: es el lugar para que el usuario reconozca lo que vio.

---

## 4. Lista de verificación antes de publicar una ficha

- [ ] ¿Un solo tema? ¿Entre 200 y 350 palabras?
- [ ] ¿El título es la pregunta del usuario, no el nombre interno?
- [ ] ¿La respuesta está en la primera línea?
- [ ] ¿Están las palabras coloquiales, con y sin acento?
- [ ] ¿El nombre del sistema coincide carácter por carácter con la base?
- [ ] ¿El enlace es completo, del canal correcto y con el casing exacto?
- [ ] ¿Dice lo que **no** se puede en este tema?
- [ ] ¿Se entiende leída aislada, sin ninguna otra ficha?
- [ ] ¿Sin datos personales, sin credenciales, sin datos que cambian solos?
- [ ] ¿Sin HTML, sin referencias internas tipo wiki, sin secuencias que puedan confundirse con delimitadores del prompt?
- [ ] ¿Tiene dueño y fecha de vigencia?

---

## 5. Ejemplo completo aplicado

> 🟨 Ilustrativo: los identificadores y requisitos son sintéticos.

```markdown
<!--
kb_id: T1-014
tipo: T1
perfil: ciudadano
fuente: lut_MotivosTurnos (Id=<n>) + sinónimos curados
dueño: Referente funcional de Turnos
vigencia: 2026-07-31
-->

## Trámite: Castración de mascotas

Sacás el turno desde el portal, en el trámite "Castración de mascotas", que
atiende la oficina de Zoonosis.

**También conocido como:** castración, castracion, castrar al perro, castrar
a la perra, capar, esterilizar, esterilización, operar al perro, castración
de gatos, turno con el veterinario, veterinaria, zoonosis, sanidad animal.

**Nombre exacto en el sistema:** Castración de mascotas
**Categoría (tipo de turno):** Zoonosis

**Detalle:**
El turno se saca a nombre tuyo, no de la mascota. Para confirmarlo te van a
pedir nombre, apellido, DNI, celular y correo electrónico: esos cinco datos
son obligatorios.

La raza, la edad y el sexo del animal no se cargan al sacar el turno; el
profesional los evalúa en la atención.

**Enlace:** https://<host>/ciudadano/TurnosLugar?m=<IdMotivo>
Ahí ves los lugares donde se atiende, los horarios disponibles y, arriba de
la lista, los requisitos del trámite.

**Lo que NO se puede:** no se puede reservar un horario por chat ni por
teléfono, y una vez sacado el turno **no se puede cambiar de fecha**: hay que
cancelarlo y sacar uno nuevo.

**Ojo:** cuando elegís el horario tenés 5 minutos para completar tus datos.
Si tardás más, el horario vuelve a quedar libre para otra persona.
```

Recuento: ~200 palabras. Entra holgada en un fragmento de 400, con margen para que el chunking nunca la parta.

---

## Documentos relacionados

[`../03-Estructura-y-Plantilla-KB.md`](../03-Estructura-y-Plantilla-KB.md) · [`A2-Checklist-Evaluacion-KB.md`](A2-Checklist-Evaluacion-KB.md) · [`../08-Glosario.md`](../08-Glosario.md)
