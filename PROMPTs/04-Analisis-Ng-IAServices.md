# Tool-Prompt — Documentación

> **Invocación**:
> - `Lee y ejecuta /IA.Prompting.Templates/PromptFramework/Examples/Notions/Ng-IAServices/04-Analisis-Ng-IAServices.md`


---

# Contexto

Lee los siguientes documentos primero:

1. `/NG/Ng-IAServices.Documentacion/Analisis/Implementacion/Antecedentes/Analisis-Asistencia-IA-ChatBotIA.md` y 
``/NG/Ng-IAServices.Documentacion/Analisis/Implementacion/Antecedentes/IA-Mercado-Libre.md`
Es un estudio sobre chatbot IA para asistencia a ususarios, definiciones, conceptos, entre otros.

2. `/NG/Ng-IAServices.Documentacion/ia-db/README.md`
Es un servicio API rest para proveer el servicio de chatbot a los diferentes sistemas tenant. Si es necesario ampliar y extraer concpetos apunta en toda la documentación de especificación del sistema `/NG/Ng-IAServices/docs`.

3. `/GDA/GDA.Core.Documentacion/ia-db/README.md`
GDA.Core es una solución con diversos sistemas con el objetivo de la gestión digital abierta de diferentes tramites de gestión de gobierno municipal, Gestión de de infracciones y multas de cara al ciudadano y la administración de funcionarios. Los proyectos de interes son en este estudio son:
3.a GDA.Core.Ciudadano: es el usuario de gestión ciudadana.
3.b GDA.Core.BackOffice.Turnos: es el panel de control de turnos por parde del funcionario de un municipio.
3.c GDA.Core.CiudadanoApp: esta es una aplicación web que se embebe en una aplicación maui mobile.

En estos sistemas estaría bueno desarrollar la asistencia de gestión de turnos como primer caso de exito a conseguir como objetivo. Tanto la asistencia de usuarios ciudadanos como tambien para los usuarios backoffice o funcionarios. Un ciudadano podría consultar si hay turno para un tramite especifico y el chatbot le podría indicar que existe ese tramite o en realidad se llama diferente e indicarle opciones y posibles enlaces hacia la página de solicitud de turno.

4. `/BD/Boleteria.Core.Documentacion/ia-db/README.md`
Es una boleteria digital para eventos como cines, show musicales, recitales, entre otros. Los proyectos de interes son en este estudio son:
4.a BoleteriaCore.Backoffice: es el panel de control para la gestión de la boletería digital. Se configuran eventos, funciones, tarifas, informes, parametros entre otros
4.b BoleteriaCore.Web: es el portal de web donde los usuarios compran sus entradas, visualizan los eventos.

En estos sistemas de boleteria digital el caso de existo objetivo a implementar sería la gestión de eventos. Que sirva de guía para usuarios inespertos en altas de eventos, funciones, tarifas. Podría indicar ante un pregunta porque el evento no se publico que configuración le faltó y donde ir. Incluso generar un enlace puntual a la página donde configurar ese parámetro que faltó.

---

# Objetivo

Estudio de integración de asistencia por IA con chatbot en sistemas de gestión digital y ventas de boletia digital. Diseño y puesta en práctica de casos de exito que sirvan de modelo para otras áreas del sistema mencionados en la sección de contexto.

---

# Solicitudes

1. Generar el plan según los objetivos dados.

2. Documentar tu investigación y propuesta en los siguientes documento markdown, dejar en el directorio `/NG/Ng-IAServices.Documentacion/Analisis/Implementacion/`: 

2.a **Ng-IAServices**
Lo que es propio del servicio de Ng-IAServices y va ser común a ambos sistemas GDA y Boleteria (como dar editar una base de conocimiento), podes mezclar ejemplos para entender los conceptos aquí, pero trata sobre metodología para crear los RAG, o las consultas dinamicas, considera todo lo necesario para montar un caso de exito nuevo. Siembra aquí recursos para facilitar la lectura de agentes IA.
2.a.1 Software Architecture Document (SAD) - único documento jearquizado en secciones 
2.a.2 High Level Design (HLD) - único documento jearquizado en secciones
2.a.3 Low Level Design - único documento jearquizado en secciones
2.a.4 Architecture Decision Record (ADR) - único documento jearquizado en secciones
2.a.5 Operations Guide	 - único documento jearquizado en secciones
2.a.6 Administrator Guide - único documento jearquizado en secciones


2.b **Asistencia sobre Turnos en GDA:**
 Aquí estará Analiza implementar como caso de exito las solicitud y gestión de turnos en GDA. Intenta diagramar un modelo en base a lo que ya se conoce en tipo de asistenca. En este sentido construye los siguientes documentos:
2.b.1 Software Architecture Document (SAD) - único documento jearquizado en secciones
2.b.2 High Level Design (HLD) - único documento jearquizado en secciones
2.b.3 Low Level Design - único documento jearquizado en secciones
2.b.4 Architecture Decision Record (ADR) - único documento jearquizado en secciones
2.b.5 Operations Guide	- único documento jearquizado en secciones
2.b.6 Administrator Guide - único documento jearquizado en secciones
2.b.7 Propone la planificicación de tareas, sprint (describe las tareas) y capacitación con referencias a los documentos generados - único documento jearquizado en secciones

2.c **Asistencia sobre Gestión de eventos/Funciones/Tarifas/parametros y Ventas en boleteria digital**:
Aquí estará el caso de exito analizado a implementar para el caso de eventos/Funciones/Tarifas/parametros.
En este sentido construye los siguientes documentos:
2.c.1 Software Architecture Document (SAD) - único documento jearquizado en secciones
2.c.2 High Level Design (HLD) - único documento jearquizado en secciones
2.c.3 Low Level Design - único documento jearquizado en secciones
2.c.4 Architecture Decision Record (ADR) - único documento jearquizado en secciones
2.c.5 Operations Guide	 - único documento jearquizado en secciones
2.c.6 Administrator Guide - único documento jearquizado en secciones
2.c.7 Propone la planificicación de tareas, sprint (describe las tareas) y capacitación con referencias a los documentos generados - único documento jearquizado en secciones


Se sumamente detallado para que se facilite la lectura por parte de humanos, usando y estimando los mejores recursos graficos que se adequeen a las descripciones, tanto de las estructura de los proyectos, snipped de código, diagramas de clases, gráficos end-to-end, diagramas de flujos, entre otros. Consulta los proyectos originales si mejora la descripción o narrativa de los texto. 

En los dos casos de exitos a evaluar suma ejemplos y preguntas que sirvan para formar las bases de conocimientos y preguntas calling según lo pedido.


---

# Framework

## Rule Set

- `/IA.Prompting.Templates/PromptFramework/RuleSets/RuleSet-Lean.md`

