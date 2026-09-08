Módulo 3 - Memoria Persistente y Contexto

En este módulo continué el proyecto del Asistente para la Gestión de Reclamos de Calidad desarrollado en los módulos anteriores.

A partir de la arquitectura multi-agente del Módulo 2, incorporé una capa de memoria híbrida para que el sistema pueda recuperar contexto entre ejecuciones y evitar perder información de una sesión.

Arquitectura

El flujo utiliza dos niveles de memoria:

Simple Memory: conserva el historial reciente de la conversación.

Airtable: funciona como memoria persistente de largo plazo vinculada a un Session_ID.

Antes de ejecutar el Manager, el workflow consulta Airtable para verificar si ya existe memoria para la sesión.

Si la sesión es nueva, se crea un registro inicial.

Si la sesión ya existe, se recuperan el resumen consolidado, el estado del caso y los datos clave.

La información recuperada se inyecta en el contexto del Manager utilizando los delimitadores:

[INICIO DE CONTEXTO COMPARTIDO]

[FIN DEL CONTEXTO COMPARTIDO]

Summarization

El flujo mantiene un contador de mensajes por sesión.

Cuando la conversación supera los 5 intercambios, se recupera el historial reciente y se utiliza un modelo económico para generar un resumen estructurado con el siguiente formato:

{
  "asunto_principal": "",
  "puntos_clave": [],
  "accion_requerida": ""
}

El resultado sobrescribe la memoria anterior sobre el mismo Session_ID, evitando crear registros duplicados.

Datos almacenados en Airtable

La tabla Memoria_Sesiones contiene:

Session_ID

Nombre_Usuario

Fecha_Actualizacion

Resumen_Consolidado

Estado_Caso

Datos_Clave

Contador_Mensajes

No se almacenan conversaciones completas, payloads, HTML ni logs técnicos.

Archivos

manager_modulo3_opisacco_ezequiel.json

PreEntrega_Modulo3_EzequielOpisacco.pdf

Los Workers utilizados por el Manager continúan siendo los desarrollados en el Módulo 2.

Proyecto

Asistente para la Gestión de Reclamos de Calidad

## Manejo de errores

El flujo contempla una vía de revisión humana cuando la solicitud no puede clasificarse correctamente.

Para la capa de memoria, si no existe un registro asociado al Session_ID, el workflow crea una memoria inicial y continúa la ejecución sin detenerse.

La actualización de memoria se realiza sobre el mismo registro asociado al Session_ID para evitar duplicados.

Como mejora futura, se contempla incorporar manejo específico de errores de conectividad con Airtable y fallos del modelo de resumen.
