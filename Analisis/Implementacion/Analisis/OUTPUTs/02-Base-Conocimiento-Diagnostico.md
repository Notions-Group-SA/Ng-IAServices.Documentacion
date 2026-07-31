---
doc_id: KB-002
doc_type: study-guide-topic
title: Diagnóstico de la base de conocimiento actual y criterios para evaluarla
status: draft
origin: ai-generated
confidence: high
owner: Administrador funcional de KB
last_review: 2026-07-31
audience: [administrador-funcional-kb, analista, referente-funcional, product-owner]
traces:
  - ../INPUTs/Concepto-Turnos.md
  - ../INPUTs/Usuarios-Turnos.md
  - ../../GDA-Turnos/02-HLD.md
  - ../../GDA-Turnos/06-Administrator-Guide.md
  - ../../../../ia-db/indexes/04_proveedores-ia-y-rag.md
---

# 02 · Diagnóstico de la base de conocimiento actual

Los dos documentos que hoy forman la base de conocimiento suman **963 palabras** y responden, entre los dos, **una** de las cinco consultas del planteo. No es un problema de volumen: es un problema de *forma*. Están escritos como manual interno para alguien que ya sabe qué es un turno, y el motor que los tiene que recuperar es léxico —compara palabras, no significados—, de modo que la brecha entre cómo escribe el manual y cómo escribe el vecino se traduce directamente en fragmentos que nunca se recuperan.

Este documento diagnostica ese corpus con evidencia medible, explica qué falla y deja las preguntas que permiten repetir el diagnóstico sobre cualquier otra KB. La forma correcta de escribirla está en [`03-Estructura-y-Plantilla-KB.md`](03-Estructura-y-Plantilla-KB.md).

---

## 1. Definición: qué es una base de conocimiento para RAG

Una KB para un asistente por IA no es documentación. Comparten materia prima, pero optimizan cosas distintas y esa diferencia decide casi todo lo demás.

| | Documentación para personas | Base de conocimiento para RAG |
|---|---|---|
| Unidad de consumo | El documento; el lector navega, busca, saltea | El **fragmento**: el motor recupera 5 y el modelo solo ve esos |
| Contexto | El lector lo arrastra de la sección anterior | Inexistente: el fragmento viaja solo |
| Vocabulario | El del dominio; el lector se adapta | El del **usuario**, o no hay recuperación |
| Referencias cruzadas | Un enlace resuelve | Un enlace es texto muerto dentro del prompt |
| Lo que no existe | Se omite | Se **declara**, o el modelo lo inventa |

🟩 En IAConnect la unidad es literal: `KnowledgeService` parte el texto en ventanas de **400 palabras con paso 350** y `RAGEngine` recupera **top-K = 5** por TF-IDF léxico, sin embeddings —`Vector_Embedding` se persiste siempre `null` ([`04_proveedores-ia-y-rag.md`](../../../../ia-db/indexes/04_proveedores-ia-y-rag.md); [`01-SAD.md` §3.3 I2, I3](../../GDA-Turnos/01-SAD.md)).

**Qué no es una KB:** no es el volcado del manual, no es la base de datos del sistema y no es el lugar donde vive información que cambia sola. Sobre esto último, ver [`07-Informacion-Dinamica.md`](07-Informacion-Dinamica.md).

---

## 2. Cómo se fragmenta hoy el corpus actual

Medición reproducible sobre los dos archivos de [`../INPUTs/`](../INPUTs/):

```bash
wc -w INPUTs/*.md
# Concepto-Turnos.md: 313 palabras
# Usuarios-Turnos.md: 650 palabras
```

Aplicando la ventana real (400 palabras, paso 350):

| Documento | Palabras | Fragmentos resultantes | Dónde corta |
|---|---|---|---|
| `Concepto-Turnos.md` | 313 🟩 | 1 | No corta |
| `Usuarios-Turnos.md` | 650 🟩 | 2 | 🟩 El primero termina en *«…y no necesariamente al horario de apertura del lugar. **Hora Max:** Corresponde al horario máximo»* — a mitad de la definición de `Hora Max` |

🟨 El corte se salva **por casualidad**: el solape de 50 palabras hace que el segundo fragmento arranque en `### Agregar nuevos turnos` y contenga la sección completa. Con 200 palabras más en el documento, el mismo mecanismo partiría la explicación de `Intervalo` y `Turnos por intervalo` en dos mitades, y ninguna de las dos sería recuperable como respuesta completa. Depender del azar del solape es exactamente lo que la regla «un tema por documento» evita.

---

## 3. Cobertura: qué de las consultas del planteo responde la KB actual

| Consulta | ¿Hay contenido? | Diagnóstico |
|---|---|---|
| C1 · castración, perro macho de 5 años, Paraná | ❌ | 🟩 `grep -c "castraci\|zoonosis\|veterinar"` sobre ambos archivos = **0**. No existe el catálogo de motivos; el asistente no puede ni confirmar que el trámite exista |
| C2 · «¿cuándo podría ser?» | ❌ | Disponibilidad = CTX-D3 (volátil). No pertenece a la KB en ningún diseño; hoy no hay tool que la resuelva |
| C3 · «¿en qué días y horarios trabajan?» | ⚠️ parcial | El corpus explica **cómo se configuran** los horarios (`Hora min`, `Hora Max`, `Intervalo`), no **cuáles son**. Responde la pregunta del funcionario, no la del vecino |
| C4 · «turno con el veterinario» | ❌ | Sinonimia: sin diccionario que conecte «veterinario» con un motivo del catálogo, el RAG léxico devuelve score 0 |
| F1 · «¿cómo agregar turnos en la agenda de zoonosis?» | ✅ parcial | 🟩 `Usuarios-Turnos.md` §«Agregar nuevos turnos» lo responde. Falta la parte de «zoonosis»: el asistente puede explicar el procedimiento genérico, no confirmar que esa oficina exista |

**Resultado: 1 de 5 respondida, 1 parcial, 3 sin cobertura.** El patrón no es aleatorio: lo que está cubierto es el conocimiento del **funcionario** (ESC-5); lo que falta es todo el del **ciudadano** (ESC-1 a ESC-4). El corpus se escribió para un público y se lo está evaluando contra otro.

---

## 4. Los ocho defectos concretos, con su consecuencia

### 4.1 Vocabulario del autor, no del usuario

`Concepto-Turnos.md` dice *«agendas de diferentes Oficinas/Profesionales las cuales pueden ser configuradas por los administradores desde el BackOffice»*. Un vecino que escribe *«quiero un turno para castrar al perro»* aporta los términos `quiero`, `turno`, `castrar`, `perro`. De esos, solo `turno` existe en el corpus —y es la palabra más frecuente del documento, con lo que su peso IDF es mínimo—. 🟩 El TF-IDF descarta además todo token de ≤2 caracteres y ~57 stop-words del español ([`01-SAD.md` §3.3 I4](../../GDA-Turnos/01-SAD.md)).

**Consecuencia:** el fragmento no se recupera, el modelo responde sin contexto y el resultado es genérico o inventado. Es el defecto más caro y el más invisible: nadie recibe un error.

### 4.2 Un documento, muchos temas

`Usuarios-Turnos.md` cubre en 650 palabras: edificios, oficinas, tipos de turno, motivos, configuración de agendas, validaciones por presentismo y turnos internos. Siete temas.

**Consecuencia:** el top-K = 5 trae fragmentos que mezclan temas, el presupuesto de contexto se gasta en material irrelevante y —cuando el documento crezca— el chunking parte ideas por la mitad.

### 4.3 Audiencias mezcladas en un corpus único

Ambos archivos son de perfil funcionario, pero se los está evaluando contra consultas de ciudadano. 🟩 IAConnect segmenta la recuperación **por tenant** (`GetListByIdTenantAsync`), no por rol dentro de un tenant: el filtro por perfil solo existe si se lo modela como tenants separados.

**Consecuencia:** si mañana se sube contenido de vecino al mismo tenant, un vecino puede recibir instrucciones de backoffice —y a la inversa, el funcionario recibe explicaciones en segunda persona que no aplican a su pantalla.

### 4.4 Enlaces que no sobreviven al prompt

🟩 `Usuarios-Turnos.md` usa sintaxis de wiki: `[[#Configuración inicial]]` y `[[Imple. Turnos|mesa de ayuda del proveedor]]`.

**Consecuencia:** dentro del prompt eso es texto literal sin destino. El modelo puede recitarlo («ver Imple. Turnos») y el usuario no tiene dónde ir. Un enlace útil en una KB es una **URL completa**, no una referencia interna.

### 4.5 Ningún deep-link

No hay una sola ruta de la aplicación en todo el corpus.

**Consecuencia:** el asistente describe pantallas en vez de llevar a ellas. 🟦 Duplica en la KB información que ya vive en la UI y que se desactualiza en el próximo rediseño, y produce respuestas largas donde bastaba un enlace ([antecedente §E·iv](../../Antecedentes/Analisis-Asistencia-IA-ChatBotIA.md)).

### 4.6 No se declara lo que el sistema **no** hace

El corpus no menciona que no existe reprogramación, ni los topes por período, ni la penalización por ausentismo, ni la reserva blanda de 5 minutos.

**Consecuencia:** la alucinación por omisión. 🟩 El sistema no tiene función de reprogramar (grep global = 0 hits) y el modelo, ante el silencio del corpus, ofrece un botón inexistente. 🟨 Este es el arquetipo del fallo que el usuario detecta recién en el mostrador.

### 4.7 Sin metadatos de vigencia ni dueño

Ningún archivo declara fecha, versión, responsable ni de qué fuente se derivó.

**Consecuencia:** no se puede responder «¿de cuándo es este dato?» ni «¿quién lo revisa?». 🟩 IAConnect guarda `Documento_Origen`, `Indice_Fragmento` y `Fecha_Alta` por fragmento, lo que permite auditar el origen —pero no suple la falta de un dueño y una cadencia de revisión.

### 4.8 Un vacío de ingesta que agrava todo lo anterior

🟩 El catálogo de endpoints de IAConnect expone `POST` y `GET` de knowledge, **no un `DELETE`** ([`03_api-endpoints.md`](../../../../ia-db/indexes/03_api-endpoints.md)), y 🟩 recargar un documento **duplica** los fragmentos porque no hay borrado previo ni dedupe por `Documento_Origen` ([`01-SAD.md` §3.3 I6](../../GDA-Turnos/01-SAD.md)).

**Consecuencia:** corregir un texto sin purgar antes deja conviviendo la versión vieja y la nueva, ambas recuperables, y el modelo puede citar la equivocada. Mientras no exista el endpoint de borrado, la purga exige acceso directo a la base de datos. Es una **precondición operativa**, no un detalle: sin ella, el ciclo de mejora del corpus degrada la KB en lugar de mejorarla.

---

## 5. Criterios de calidad: cómo se distingue una KB buena de una pobre

| Dimensión | Versión pobre | Versión buena |
|---|---|---|
| **Recuperabilidad** | Usa el vocabulario del sistema | Siembra el vocabulario del usuario en el propio texto |
| **Atomicidad** | Un documento, muchos temas | Un tema por documento, 300–350 palabras |
| **Autosuficiencia** | El fragmento necesita al vecino para entenderse | Se entiende leído aislado |
| **Accionabilidad** | Describe la pantalla | Da la URL exacta que emite el código |
| **Honestidad** | Solo dice lo que el sistema hace | Declara explícitamente lo que **no** hace |
| **Literalidad** | Parafrasea los mensajes de error | Los cita textualmente, como el usuario los vio |
| **Segmentación** | Un corpus para todos los públicos | Un corpus por perfil, con el común generado desde una única fuente |
| **Gobierno** | Sin dueño ni fecha | Dueño nombrado, cadencia fija, versionado en Git |

🟨 Prueba práctica que ordena todo esto: **tomá 20 fragmentos al azar y leelos aislados**. Si uno no se entiende sin su vecino, el chunking está partiendo ideas. Si ninguno contiene las palabras con las que un usuario real preguntaría, la KB no es recuperable por más correcta que sea.

---

## 6. Preguntas guía

Las que hay que responder para saber si una base de conocimiento es adecuada. La versión operativa, con casillas, está en [`Anexos/A2-Checklist-Evaluacion-KB.md`](Anexos/A2-Checklist-Evaluacion-KB.md).

**Sobre el contenido**

1. Tomá diez preguntas reales de usuarios —de mesa de ayuda, del call center, del buzón de contacto—. ¿Cuántas responde el corpus hoy? Si no tenés diez preguntas reales, ese es el primer problema, no la KB.
2. Para cada pregunta, ¿las palabras que usó el usuario aparecen literalmente en algún fragmento? ¿Y sus variantes coloquiales, con y sin acento?
3. ¿Existe un documento que enumere lo que el sistema **no** puede hacer? Si no existe, ¿dónde va a buscar el modelo cuando le pregunten por eso?

**Sobre la forma**

4. ¿Cuántos temas cubre el documento más largo? ¿Cuántos fragmentos genera y por dónde corta exactamente?
5. ¿Cada fragmento se entiende leído solo, sin el anterior?
6. ¿Los enlaces son URLs completas y exactas —incluido el casing de los parámetros— o referencias internas que no llevan a ningún lado?

**Sobre el gobierno**

7. ¿Quién es el dueño de este contenido, con nombre? ¿Cada cuánto lo revisa?
8. ¿Qué pasa cuando alguien da de baja un trámite en el ABM? ¿Algo avisa a la KB, o queda mintiendo en silencio?
9. ¿Cómo se corrige un texto ya cargado sin duplicar fragmentos?
10. ¿Existe un banco de preguntas de regresión que se corra antes de publicar un cambio del corpus?

**Sobre los límites**

11. ¿Hay algún dato en la KB que cambie sin intervención de un editor? Si lo hay, está mal ubicado.
12. ¿Hay algún dato personal en la KB? Si lo hay, es un incidente, no una mejora pendiente.

---

## 7. Anexo — Diagnóstico aplicado, en una tabla

Resumen del estado del corpus actual contra los criterios de §5. Es el formato con el que conviene entregar cualquier diagnóstico de KB.

| Criterio | `Concepto-Turnos.md` | `Usuarios-Turnos.md` | Evidencia |
|---|---|---|---|
| Recuperabilidad | ❌ | ⚠️ | 🟩 0 coincidencias de castración/zoonosis/veterinario |
| Atomicidad | ⚠️ 3 temas en 313 palabras | ❌ 7 temas en 650 | Lectura de los archivos |
| Autosuficiencia | ✅ (1 fragmento) | ⚠️ el corte cae en `Hora Max` | Cálculo de ventana 400/350 |
| Accionabilidad | ❌ | ❌ | 0 URLs |
| Honestidad (límites) | ❌ | ❌ | Ninguna mención de lo no soportado |
| Literalidad de mensajes | ❌ | ❌ | Ningún mensaje del sistema citado |
| Segmentación por perfil | ❌ ambos son de perfil funcionario en un corpus único | Estructura de `INPUTs/` |
| Gobierno | ❌ | ❌ | Sin frontmatter, fecha ni dueño |

🟨 Ninguno de estos defectos es un error de quien redactó: los dos archivos son buena documentación interna. El desajuste es de **destino**. Convertir documentación en corpus es un trabajo de reescritura, no de copia —y es exactamente lo que resuelve el documento siguiente.

---

## Documentos relacionados

[`00-Marco-Referencia.md`](00-Marco-Referencia.md) · [`03-Estructura-y-Plantilla-KB.md`](03-Estructura-y-Plantilla-KB.md) · [`04-Metodologias-y-Catalogacion.md`](04-Metodologias-y-Catalogacion.md) · [`07-Informacion-Dinamica.md`](07-Informacion-Dinamica.md) · [`Anexos/A2-Checklist-Evaluacion-KB.md`](Anexos/A2-Checklist-Evaluacion-KB.md)
