---
doc_id: GLO-008
doc_type: study-guide-glossary
title: Glosario de asistencia conversacional por IA aplicada a sistemas de gestión
status: draft
origin: ai-generated
confidence: high
owner: Analista de la guía (NG-SA)
last_review: 2026-07-31
audience: [todos]
traces:
  - ../01-Planteo-Analisis-Contexto.md
  - ../../Antecedentes/Analisis-Asistencia-IA-ChatBotIA.md
  - ../../Antecedentes/IA-Mercado-Libre.md
  - ../../GDA-Turnos/01-SAD.md
---

# 08 · Glosario

Definiciones operativas: cada entrada dice qué es el término, para qué sirve y —cuando corresponde— cómo se materializa en IAConnect o en GDA. Los términos que el planteo pidió explícitamente están desarrollados en §1; el resto del vocabulario de la guía, en §2, ordenado alfabéticamente.

---

## 1. Los tres términos del planteo

### Contenido curado

**Definición.** Conjunto de contenido **seleccionado, reescrito y mantenido a propósito** para que lo consuma un asistente, en oposición a contenido que simplemente existe en la organización y se vuelca al índice tal como está. Curar implica cuatro decisiones deliberadas: qué entra, cómo se redacta, con qué acción se asocia y quién lo mantiene.

**Qué lo distingue de «documentación indexada».** No es una cuestión de calidad de la redacción original, sino de destino:

| Contenido volcado | Contenido curado |
|---|---|
| Se indexa lo que ya estaba escrito | Se decide qué merece estar y se reescribe |
| Vocabulario del autor | Vocabulario del usuario, sembrado en el texto |
| Dice lo que el sistema hace | Dice también lo que **no** hace |
| Sin acción asociada | Cada ficha lleva su enlace o su siguiente paso |
| Sin dueño | Dueño nombrado y cadencia de revisión |

**Sobre la expresión «Mercado Pago apunta a contenido curado».** 🟨 Es una **inferencia** de esta guía a partir de lo observable en las capturas analizadas, no un dato interno de la empresa. Los indicios son de forma: el asistente declara sus propios límites (algo que un artículo de ayuda no suele hacer), corrige supuestos frecuentes del usuario (alguien conocía ese supuesto antes de escribir la respuesta) y entrega la acción asociada mediante deep-link (dato de producto, no de documentación). De ahí se deduce una práctica de curaduría; **no** se puede afirmar qué herramienta usan, cómo indexan ni con qué frecuencia reindexan. Desarrollo completo en [`04-Metodologias-y-Catalogacion.md` §3](04-Metodologias-y-Catalogacion.md).

Ver también: [[Corpus]], [[Base de conocimiento (KB)]], [[Golden set]].

---

### Deep-link

**Definición.** Enlace que lleva directamente a una **pantalla concreta de una aplicación con su contexto ya cargado**, en lugar de a la portada. En un asistente conversacional es el mecanismo de *hand-off*: en vez de describir cinco pasos, se entrega el enlace que ejecuta el primero.

**Por qué importa acá.** Dos razones, y la segunda suele pasarse por alto:

1. **Atención.** En un chat —sobre todo en un celular— un bloque largo se saltea, y el usuario vuelve a preguntar algo que ya estaba respondido.
2. **Mantenimiento.** Describir una pantalla paso a paso **duplica en la KB** información que ya vive en la interfaz, y ese duplicado se desactualiza en el próximo rediseño sin que nadie se entere. El deep-link no tiene esa deuda: delega en la pantalla la responsabilidad de estar al día.

**Trampas verificadas en GDA.** 🟩 Las rutas **no son intercambiables entre canales**: el portal ciudadano usa `PathBase` `/ciudadano` y la app `/`, y cada uno tiene páginas que el otro no tiene. 🟩 Hay diferencias de mayúsculas en los parámetros de consulta (`TurnoDetalle?Id=` con `I` mayúscula) y 🟩 typos en rutas públicas que **no deben corregirse**, porque romperían enlaces del wrapper nativo. Regla: el corpus lleva la ruta **exactamente como la emite el código**, por canal.

🟨 Regla de seguridad asociada: en cuanto existan tools, el deep-link debe **devolverlo la tool**, nunca construirlo el modelo, y validarse contra una lista blanca de rutas. Un enlace emitido por un LLM es contenido no confiable: una injection en el corpus alcanzaría para dirigir usuarios a un destino externo con apariencia legítima.

Ver también: [[Hand-off]], [[Divulgación progresiva]], [[Insecure output handling]].

---

### Corpus

**Definición.** El **conjunto completo de textos indexados** de un asistente, considerado como una unidad. No es «los documentos»: es el material efectivamente cargado, fragmentado e indexado, que es lo único que el motor de recuperación puede alcanzar.

**Por qué se lo nombra aparte de «documentación».** Porque tiene propiedades propias que se miden sobre el conjunto, no sobre cada documento:

| Propiedad del corpus | Qué mide | Valor de referencia en este caso |
|---|---|---|
| **Tamaño** | Cantidad de fragmentos | 🟨 ~50–70 por tenant: 🟩 `RAGEngine` carga todos los fragmentos en memoria por request |
| **Cobertura** | Qué proporción de las preguntas reales tiene respuesta | Se mide con el golden set |
| **Densidad léxica** | Si contiene las palabras que los usuarios usan | Se mide con consultas sin resultado relevante |
| **Frescura** | Distancia entre el corpus y el sistema real | Sin job de verificación, es desconocida |
| **Segmentación** | Cómo se particiona por público | 🟩 En IAConnect, por `Id_Tenant` |

🟨 De esas cinco, la que sorprende es el tamaño: **un corpus más grande puede responder peor**. Más fragmentos diluyen la señal léxica, agregan latencia y aumentan la probabilidad de que los cinco recuperados sean los equivocados. El objetivo no es cubrir todo el dominio, es cubrir lo que se pregunta.

**En IAConnect el corpus es, literalmente,** 🟩 las filas de `sys_Fragmentos_Conocimiento` de un tenant, cada una con `Documento_Origen`, `Indice_Fragmento`, `Contenido` y `Fecha_Alta`.

Ver también: [[Chunk / fragmento]], [[RAG]], [[Contenido curado]], [[Tenant]].

---

## 2. Vocabulario de la guía

| Término | Definición |
|---|---|
| **Asistente por IA** | Chatbot construido sobre un LLM que comprende la intención, se apoya en conocimiento y datos —no solo en reglas— y puede ejecutar acciones sobre el sistema anfitrión |
| **Base de conocimiento (KB)** | El contenido escrito que alimenta al asistente, antes de ser fragmentado. Su forma indexada es el [[Corpus]] |
| **Chatbot** | Cualquier sistema que conversa en lenguaje natural, con o sin LLM detrás |
| **Chunk / fragmento** | Porción de un documento indexada como unidad de recuperación. 🟩 En IAConnect: ventana de 400 **palabras** con paso 350 |
| **Chunking** | Proceso de partir un documento en fragmentos. El solapamiento evita que una idea quede sin representante íntegro en el índice |
| **Contexto (ventana de)** | Cantidad máxima de texto que el modelo procesa por invocación. Lo consumen el system prompt, los fragmentos, el historial y la consulta |
| **CTX-D1…D4** | Categorías de naturaleza del dato usadas en esta guía: estable, semi-estable, volátil, personal. Ver [`00 §2.3`](00-Marco-Referencia.md) |
| **Deep-link** | Ver §1 |
| **Desambiguación** | Turno conversacional en el que el asistente pide o propone precisión ante un pedido con varios candidatos, **sin reiniciar el contexto** |
| **Disclosure de alcance** | Declaración explícita de lo que el asistente **no** puede ver o hacer, en lugar de improvisar una respuesta. Reduce además la presión de jailbreak |
| **Divulgación progresiva** | Entregar el paso necesario ahora y ofrecer el resto si se pide, en lugar de volcar el procedimiento completo |
| **Embedding** | Vector que representa el significado de un texto y habilita la búsqueda semántica. 🟩 En IAConnect la columna existe (`Vector_Embedding`) pero se persiste siempre `null` |
| **Entity / slot** | Dato que hay que capturar para resolver una intención (motivo, fecha, oficina). Los slots sensibles —identidad— se toman de la sesión, nunca del texto |
| **Fase (F1/F2/F3)** | Nivel de capacidad del asistente: informativo, con datos en vivo, transaccional |
| **Function-calling / tools** | Mecanismo por el cual el LLM invoca una API con un esquema declarado para obtener datos o ejecutar acciones. 🟩 No existe en IAConnect |
| **Golden set** | Conjunto fijo de preguntas con su respuesta esperada, que se ejecuta ante cada cambio del corpus para detectar regresiones |
| **Groundedness** | Grado en que la respuesta está efectivamente sostenida por los fragmentos recuperados. Es la métrica anti-alucinación |
| **Guardrails** | Controles de entrada y salida que acotan alcance y riesgo: anti-injection, límites de tamaño, validación de salida |
| **Hand-off** | Derivación a un humano o a un flujo nativo del sistema cuando el caso excede el alcance del asistente |
| **Idempotencia** | Propiedad por la cual repetir una operación no produce un efecto adicional. Requisito no negociable de toda acción ejecutada desde un chat, donde el reintento es normal |
| **Insecure output handling** | Clase de vulnerabilidad: ejecutar o renderizar la salida del modelo sin validarla (HTML crudo, enlaces, parámetros) |
| **Intent** | Intención del usuario, catalogada con identificador estable. Los **intents negativos** catalogan lo que el usuario pedirá y el sistema no hace |
| **LLM** | Modelo de lenguaje grande; el motor generativo del asistente |
| **Multi-tenant** | Arquitectura en la que varios clientes u organizaciones comparten la plataforma con sus datos aislados. 🟩 IAConnect aísla por `Id_Tenant` en API, sesiones y corpus |
| **Prompt injection** | Ataque que introduce instrucciones maliciosas por la entrada del usuario (directa) o por un documento que el sistema recupera (**indirecta**, la más difícil de detectar) |
| **RAG** *(Retrieval-Augmented Generation)* | Recuperar fragmentos propios y añadirlos al prompt para que el modelo responda sobre conocimiento de la organización |
| **RAG léxico / semántico / híbrido** | Recuperación por coincidencia de términos (TF-IDF, BM25), por significado (embeddings) o por ambos con re-ranking. 🟩 IAConnect es léxico |
| **Re-ranking** | Reordenamiento de los candidatos recuperados por un segundo criterio, para mejorar la precisión del top-K |
| **Siembra léxica** | Práctica de escribir dentro de la ficha las palabras coloquiales con las que el usuario preguntaría, para que un RAG léxico pueda recuperarla |
| **Sinónimos (diccionario de)** | Documento curado que traduce el vocabulario del usuario al del sistema. 🟦 Es *query expansion* por tesauro, técnica anterior a los LLM y vigente donde la recuperación es léxica |
| **Stop-words** | Palabras muy frecuentes que el motor descarta por no aportar señal. 🟩 En IAConnect: ~57 del español, más todo token de ≤2 caracteres |
| **System prompt** | Instrucción inicial que define rol, tono, dominio y reglas del asistente. 🟩 En IAConnect es configurable por tenant y viaja entero en cada request |
| **Tenant** | Cliente u organización aislada dentro de la plataforma. Acá se usa además como unidad de segmentación por **perfil**, porque es el único filtro disponible |
| **TF-IDF** | Medida clásica de relevancia léxica: pondera la frecuencia de un término en el documento contra su rareza en el corpus |
| **Top-K** | Cantidad de fragmentos que la recuperación devuelve. 🟩 En IAConnect, 5 fijo |
| **Trazabilidad / cita de origen** | Registrar de qué documento salió cada afirmación. 🟩 `Documento_Origen` e `Indice_Fragmento` lo habilitan |

---

## 3. Alias y equivalencias

Términos que aparecen indistintamente en la documentación del proyecto y que designan lo mismo:

| Canónico | Alias en uso |
|---|---|
| Corpus | base de conocimiento indexada, KB indexada |
| Chunk | fragmento |
| Function-calling | tools, herramientas, `tool_use` |
| Deep-link | enlace profundo, link directo |
| Disclosure de alcance | declaración de límites, aviso de alcance |
| Hand-off | derivación, escalamiento |
| Golden set | banco de preguntas de regresión |

---

## Documentos relacionados

[`00-Marco-Referencia.md`](00-Marco-Referencia.md) · [`04-Metodologias-y-Catalogacion.md`](04-Metodologias-y-Catalogacion.md) · [`07-Informacion-Dinamica.md`](07-Informacion-Dinamica.md) · [`../../Antecedentes/Analisis-Asistencia-IA-ChatBotIA.md`](../../Antecedentes/Analisis-Asistencia-IA-ChatBotIA.md)
