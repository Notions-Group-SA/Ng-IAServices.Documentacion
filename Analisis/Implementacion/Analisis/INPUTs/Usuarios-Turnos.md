Este manual hace referencia a como configurar las distintas opciones del apartado de Turnos
## Configuración inicial

### Edificios y oficinas

El paso 1 y paso 2 corresponde a la configuración de los lugares físicos de atención al vecino. Esto servirá para quien parametricé mediante BackOffice.

**PASO 1: Edificios**
Se deben cargar o editar los edificios (lugares físicos) de las oficinas donde se administraran las agendas de turnos. Por ejemplo Hospital Municipal o Tránsito.

**PASO 2: Oficinas**
Crear o editar las oficinas correspondientes al edificio creado anteriormente. Estas representan al lugar o profesional que administrara las agendas. Por ejemplo licencia de conducir dentro del edificio tránsito o clínica medica dentro del edificio Hospital Municipal. También puede ser el nombre del profesional, como por ejemplo Dr. Juan Pérez en el edificio Hospital Municipal.
Aquí podrán parametrizar el rango horario y cantidad de turnos por usuario.

### Tipos y motivos de turnos

Los pasos siguientes corresponden a la configuración de como los vecinos van a buscar el tipo y motivo de turno por el cual desean ser atendidos.

**PASO 3: Tipos de turnos**
Los tipos de turno son una forma de agrupación por concepto. Por ejemplo un tipo de turno podría ser "Salud" o "Licencia de conducir".

**PASO 4: Motivos de turnos**
El motivo de turno es la mínima expresión del trámite o concepto que la persona desea sacar un turno. Siguiendo con los ejemplos anteriores, dentro del Tipo de Turno "Salud", podríamos crear los motivos: Clínica medica, oftalmología, etc.
En esta sección determinamos cuales son las oficinas en donde la persona puede realizar el "Motivo de turno" Es decir, dentro del motivo "Clínica medica" puede ser que esta especialidad se pueda atender en diferentes oficinas, incluso de edificios diferentes.

## Configuración de agendas

Una vez parametrizados las oficinas y todo lo expresado en la [[#Configuración inicial]] ingresar a "Agendas Turnos" y elegir la oficina que se desea configurar.
Inicialmente el sistema muestra un resumen de la configuración de la oficina elegida. allí verán desde y hasta que fecha se quiere cargar turnos, así como también cual fue la fecha del ultimo turno tomado.

### Agregar nuevos turnos

Una vez seleccionada la oficina, ingresamos con el botón "agregar turnos". Ahí completamos toda la codificación que nos pide el sistema. 

A tener en cuenta:
**Hora min:** Corresponde al primer horario que el sistema va a entregar el turno y no necesariamente al horario de apertura del lugar.
**Hora Max:** Corresponde al horario máximo de atención. Por ejemplo si tenemos un intervalo de 30 minutos y ponemos en hora máxima las 13hs. el sistema va a entregar el último turno a las 12:30hs
**Intervalo:** Corresponde a cada cuantos minutos se atiende un turno
**Turnos por intervalo:** Esto te permite configurar la cantidad de turnos por horario. Esto sirve para los casos en donde se quiere que por ejemplo se atiendan 3 personas en cada horario.

# Validaciones por presentismo y/o cantidad de turnos

El sistema permite configurar a partir de que cantidad de ausencias un ciudadano no puede sacar mas turnos y debe solicitarlos a la mesa de ayuda del municipio.
También se puede limitar la cantidad de turnos futuros que un ciudadano puede sacar, independientemente de si tiene ausencias o no. 
Esto no esta disponible en el módulo de parámetros, se debe solicitar a la [[Imple. Turnos|mesa de ayuda del proveedor]].

# Turnos internos

Puede darse la necesidad de un cliente que los turnos solo puedan darse desde la ofician y no cualquier persona pueda sacar un turno. Para ello se parametriza de la siguiente manera.
En el módulo de parámetros, ingresar a la sección turnos y luego oficina de atención. Ingresar a la que se desee editar y activa la opción "Interno". Esto lo que hace es que todos los turnos solo se puedan otorgar desde el BackOffice Turno y solo por el personal habilitado que administre las agendas de esa oficina.