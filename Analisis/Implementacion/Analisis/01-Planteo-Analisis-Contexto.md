## Crear prompt Analisis - Análisis sobre contexto e implementación de chatbot IA.

> **Invocación**:
> - `Lee y ejecuta /NG/Ng-IAServices.Documentacion\Analisis/Implementacion/Analisis/01-Planteo-Analisis-Contexto.md`
> **Overview**: Análisis sobre contexto e implementación de chatbot IA.

---

## Contexto

Imagina que a un Chatbot-IA se le hacen las siguientes preguntas:

### **Preguntas típicas de usuarios consumidores del sistema (ciudadanos)**

  Consulta ciudadano 1: 
  > `Hola, como estás? Me gustaría saber si hay turno para castración. Es para mi perrito, macho, raza salchicha, tiene 5 años. Soy de Paraná.`

  Consulta ciudadano 2: 
> `Vengo a sacar turno para castrar al perro, cuándo podría ser?`

  Consulta ciudadano 3: 
> `Buenas! Quería solicitar un turno para castración, en que días y horarios trabajan ? Gracias`

  Consulta ciudadano 4:
>**Ciudadano:** ¿Quiero sacar un turno con para el veterinario?

### Usuario administrador del sistema (funcionario): 

  Consulta agente municipal 1: `
> `¿Cómo agregar turnos en la agenda en la oficina de zoonosis?`

### Base de conocimiento actual 

  Mi RAG o base de conocimiento es básico todavía , pero tengo el siguiente contenido como base de conocimiento:
    - `/NG/Ng-IAServices.Documentacion/Analisis/Implementacion/Analisis/INPUTs/Concepto-Turnos.md`
    - `/NG/Ng-IAServices.Documentacion/Analisis/Implementacion/Analisis/INPUTs/Usuarios-Turnos.md`

  Ahora, si es cierto que esa base de conocimiento no alcanza para dar respuestas a todas esas preguntas, pero en base como responde el **Chatbot-IA** de mercado pago, y según lo que sabemos: `/NG/Ng-IAServices.Documentacion/Analisis/Implementacion/Antecedentes/Analisis-Asistencia-IA-ChatBotIA.md`.

  Y por otro lado tenemos que en `/GDA/GDA.Core.Documentacion/ia-db/README`, lo relativo a `GDA.Core.BackOffice.Turnos`, que sería el `panel de control de turnos para funcionarios` y en lo relativo a `GDA.Core.Ciudadano` y `GDA.Core.CiudadanoApp` , tal que los cuales ambos dos son las aplicaciones de cara al usuario ciudadano.

  Más también, tengo el servicio de chat `/NG/Ng-IAServices.Documentacion/ia-db/README.md`, que trata sobre una API para Chatbot-IA denominado: `IAConnect`.

  Donde, los documentos en los que se realizó un análisis previo están en: `/NG/Ng-IAServices.Documentacion/Analisis/Implementacion/GDA-Turnos` 

---
## Objetivos

- Tener un panorama de que es Chatbot-IA, que implica , formar un criterio de discernimiento de dicha tecnología, y contar con un contexto general e inicial para su implantación suficiente para entender como integrarlo en el sistema de turnos. 

---
## Solicitudes

Pero en función de este contexto surgen cuestiones interesantes a evaluar
1. ¿La base de conocimientos actual si está redactada correctamente?. ¿Cuál sería la estructura correcta, forma del relato, preguntas que me debería hacer para saber si es adecuada o preguntas guías para elaborar una mejor base de conocimientos?. Elaborar un plantilla adecuada genérica para estructurar mi base de conocimientos. ¿Qué presupones que usa mercado pago para la confección de su base de conocimientos?. Qué metodologías se conocen. ¿Se catalogan las preguntas?

2. En función a la funcionalidad y aspectos técnicos. ¿Qué debería tener en cuenta en `IAConnect` para integrarlo en `panel de control de turnos para funcionarios`, y luego para integrarlo en las aplicaciones destinada a los usuarios ciudadanos? , Ahora, que alcances tiene para ser implementado en su estado de desarrollo actual, principalmente en funcionarios, que aspecto técnicos/funcionales faltaría para cumplir con la funcionalidad que tiene el Chatbot-IA de mercado pago.

3. En base a las posibles preguntas dadas como ejemplo en la sección contexto, teniendo en cuenta que el Chatbot-IA tiene acceso a la base de datos o sistema para determinar posibles turnos disponibles. ¿Qué flujo conversacional podría emitir para cada caso presentador? ¿Podría el Chatbot-IA reservar un turno? o simplemente diría los próximos turnos disponibles, Que alcance tendría.

4. Respecto al desarrollo de hoy de `IAConnect`, actualmente, haceme un documento que se focalice solo en: 
- ¿Para responder preguntas respecto a como hacer tal cosa o cual cosa? se que tengo la base de conocimientos para que el Chatbot-IA responda esas consultas. Ahora para las preguntas relacionadas con la información cambiante, o la información propia del usuario que se encuentra en el usuario, o bien por ejemplo, cambios en la agenda de turnos, ¿cómo la resuelvo a hora como esta en su estado de desarrollo y/o cómo  la actualizo?, analiza la situación.

5. Arma un glosario completo, con definiciones. Incluí los siguientes términos
  - ¿A qué se hacer referencia con?
  - en referencia a `...mercado pago apunta a contenido curado ...`
  - Deep-Link
  - Corpus 
 
6. Crear los documentos necesarios en el destino: `/NG/Ng-IAServices.Documentacion/Analisis/Implementacion/Analisis/OUTPUTs` que resuelvan y expliquen las cuestiones planteadas.



---

## Reglas
- Los documentos generados ubícalos en `/NG/Ng-IAServices.Documentacion/Analisis/Implementacion/Analisis/OUTPUTs`. Definiciones, ejemplos, preguntas guías con respuesta explicando sus posibles respuestas. 
- No inventar información.
- Toda afirmación deberá estar respaldada por evidencias verificables.


---

## Framework


### Profile

  Aplicar:
- `/IA/IA.Prompts/PromptFramework/Profiles/Study-Guide-Documentation.md`


