---
doc_id: ANX-A2
doc_type: checklist
title: Lista de verificación para evaluar una base de conocimiento
status: draft
origin: ai-generated
confidence: high
owner: Administrador funcional de KB
last_review: 2026-07-31
audience: [administrador-funcional-kb, analista, referente-funcional, qa]
traces:
  - ../02-Base-Conocimiento-Diagnostico.md
  - ../03-Estructura-y-Plantilla-KB.md
  - ../07-Informacion-Dinamica.md
---

# A2 · Lista de verificación de una base de conocimiento

Checklist aplicable a cualquier corpus destinado a un asistente por IA. Se responde con evidencia, no con impresión: cada ítem incluye cómo verificarlo. El diagnóstico del corpus actual, hecho con esta misma lista, está en [`02-Base-Conocimiento-Diagnostico.md` §7](../02-Base-Conocimiento-Diagnostico.md).

**Criterio de corte:** un ítem marcado ❌ en las secciones 1, 4 o 6 bloquea la publicación. El resto son deudas registrables.

---

## 1. Cobertura — ¿responde lo que se pregunta?

| # | Verificación | Cómo se comprueba | Estado |
|---|---|---|---|
| 1.1 | Existe un listado de al menos 20 preguntas **reales** de usuarios | Mesa de ayuda, mostrador, buzón de contacto, logs del asistente | ☐ |
| 1.2 | Cada pregunta del listado tiene una ficha que la responde, o está marcada como hueco conocido | Recorrido manual pregunta × corpus | ☐ |
| 1.3 | El listado incluye preguntas que el sistema **no** puede resolver | Revisión con el referente funcional | ☐ |
| 1.4 | Existe una ficha de límites (T7) | Búsqueda del documento en el corpus | ☐ |
| 1.5 | Ninguna pregunta se responde con dos fichas contradictorias | Búsqueda de duplicados por tema | ☐ |

---

## 2. Recuperabilidad — ¿el motor lo encuentra?

| # | Verificación | Cómo se comprueba | Estado |
|---|---|---|---|
| 2.1 | Las palabras coloquiales del usuario aparecen literalmente en las fichas | `grep` de los cinco términos más frecuentes de cada consulta real | ☐ |
| 2.2 | Las variantes con y sin acento están escritas cuando el dato real va sin tilde | `grep` de ambas formas | ☐ |
| 2.3 | Existe un diccionario de sinónimos (T2) si la recuperación es léxica | Búsqueda del documento | ☐ |
| 2.4 | Los términos críticos sobreviven al filtro del motor (>2 caracteres, no stop-word) | Revisión contra la lista de stop-words del motor | ☐ |
| 2.5 | Prueba en vivo: 10 consultas reales devuelven la ficha esperada entre los primeros resultados | Ejecución contra el asistente | ☐ |

---

## 3. Forma — ¿sobrevive al fragmentado?

| # | Verificación | Cómo se comprueba | Estado |
|---|---|---|---|
| 3.1 | Ninguna ficha supera el tamaño de la ventana de chunking | `wc -w` sobre cada archivo | ☐ |
| 3.2 | Cada documento trata **un** tema | Lectura del índice de encabezados | ☐ |
| 3.3 | Prueba de aislamiento: 20 fragmentos al azar se entienden leídos solos | Lectura ciega por alguien ajeno al corpus | ☐ |
| 3.4 | No hay referencias internas («ver la sección anterior», wikilinks `[[…]]`) | `grep` de patrones de referencia | ☐ |
| 3.5 | Los enlaces son URLs completas, del canal correcto y con el casing exacto | Comparación contra las rutas del código | ☐ |
| 3.6 | Los mensajes del sistema están citados literalmente | Comparación contra el código o la UI | ☐ |

---

## 4. Naturaleza del dato — ¿está en el lugar correcto?

| # | Verificación | Cómo se comprueba | Estado |
|---|---|---|---|
| 4.1 | Cada fuente está clasificada como estable / semi-estable / volátil / personal | Tabla de clasificación completa | ☐ |
| 4.2 | **No hay datos volátiles en el corpus** (disponibilidad, cupos, estados) | Revisión dirigida | ☐ |
| 4.3 | **No hay datos personales en el corpus** | `grep` de patrones de DNI, teléfono, correo | ☐ |
| 4.4 | Los datos semi-estables tienen dueño y fecha de vigencia declarados | Revisión del encabezado de cada ficha | ☐ |
| 4.5 | Está escrito el texto de disclosure para cada dato que el asistente **no** puede ver | Búsqueda en el corpus y en el system prompt | ☐ |

---

## 5. Segmentación — ¿cada público ve lo suyo?

| # | Verificación | Cómo se comprueba | Estado |
|---|---|---|---|
| 5.1 | Los perfiles con permisos o necesidades distintas tienen corpus separados | Inventario por tenant | ☐ |
| 5.2 | El contenido común se genera desde una única fuente versionada | Revisión del pipeline de ingesta | ☐ |
| 5.3 | La voz corresponde al perfil (2.ª persona al ciudadano, 3.ª al funcionario) | Lectura de muestra | ☐ |
| 5.4 | Prueba negativa: una consulta cuya respuesta solo existe en el otro corpus no devuelve contenido | Ejecución contra el asistente | ☐ |

---

## 6. Higiene e ingesta — ¿el ciclo de actualización no lo degrada?

| # | Verificación | Cómo se comprueba | Estado |
|---|---|---|---|
| 6.1 | El corpus está versionado en un repositorio, no solo cargado en la base | Revisión del repositorio | ☐ |
| 6.2 | Existe un procedimiento de purga antes de recargar | Revisión del script de ingesta | ☐ |
| 6.3 | Recargar dos veces el mismo documento **no** duplica fragmentos | Prueba: contar fragmentos antes y después | ☐ |
| 6.4 | El contenido derivado del sistema se genera por ETL, no a mano | Revisión del pipeline | ☐ |
| 6.5 | La ingesta sanitiza HTML y secuencias que puedan confundirse con delimitadores del prompt | Revisión del ETL | ☐ |
| 6.6 | Existe un job que compara el corpus contra las tablas de origen y alerta divergencias | Búsqueda del job | ☐ |

---

## 7. Gobierno y mejora — ¿alguien se hace cargo?

| # | Verificación | Cómo se comprueba | Estado |
|---|---|---|---|
| 7.1 | Cada documento tiene un dueño con nombre | Encabezados del corpus | ☐ |
| 7.2 | Hay cadencia de revisión definida por tipo de contenido | Documento de gobierno | ☐ |
| 7.3 | Existe un golden set que se corre antes de publicar | Repositorio del set + evidencia de ejecución | ☐ |
| 7.4 | Se registran las consultas sin respuesta relevante | Instrumentación o revisión manual de conversaciones | ☐ |
| 7.5 | Ese registro alimenta un backlog de contenido | Backlog visible | ☐ |
| 7.6 | Hay una revisión periódica con responsable asignado | Calendario | ☐ |

---

## 8. Cómo se reporta el resultado

Formato recomendado de entrega. La columna de evidencia es obligatoria: un diagnóstico sin evidencia es una opinión.

| Sección | Ítems ✅ | Ítems ❌ | Bloqueantes | Evidencia |
|---|---|---|---|---|
| 1 · Cobertura | | | | |
| 2 · Recuperabilidad | | | | |
| 3 · Forma | | | | |
| 4 · Naturaleza del dato | | | | |
| 5 · Segmentación | | | | |
| 6 · Higiene e ingesta | | | | |
| 7 · Gobierno | | | | |

🟨 Un corpus nuevo rara vez supera la mitad de la lista en su primera evaluación, y eso es esperable. Lo que no es aceptable es publicar con ❌ en 4.2, 4.3 o 6.3: son los tres ítems cuyo fallo produce daño en lugar de solo un mal servicio.

---

## Documentos relacionados

[`../02-Base-Conocimiento-Diagnostico.md`](../02-Base-Conocimiento-Diagnostico.md) · [`../03-Estructura-y-Plantilla-KB.md`](../03-Estructura-y-Plantilla-KB.md) · [`A1-Plantilla-KB.md`](A1-Plantilla-KB.md) · [`../07-Informacion-Dinamica.md`](../07-Informacion-Dinamica.md)
