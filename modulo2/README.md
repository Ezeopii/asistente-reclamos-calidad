# Módulo 2 - Orquestación Multi-Agente

En esta etapa del proyecto amplié el agente base desarrollado en el Módulo 1 y lo convertí en una arquitectura multi-agente utilizando el patrón Manager-Worker.

El objetivo fue separar responsabilidades dentro del flujo para que cada componente tenga una función específica y el sistema pueda seguir creciendo de forma modular.

## Arquitectura

El flujo principal funciona como Manager y se encarga de recibir la solicitud, interpretar la intención y derivarla al especialista correspondiente.

Se configuraron dos Workers independientes:

- Worker 1 - Analizador de Reclamos  
  Analiza la información recibida e identifica los datos relevantes del reclamo.

- Worker 2 - Redactor de Comunicaciones  
  Genera borradores de comunicaciones relacionadas con el seguimiento de los reclamos.

Además, se incorporó una ruta de revisión humana para las solicitudes que no puedan clasificarse correctamente.

## Flujo general

Chat Trigger → Manager → Enrutamiento → Worker correspondiente → Log de trazabilidad

Antes de ejecutar cada Worker, el Manager prepara únicamente los datos necesarios mediante nodos Edit Fields.

Los sub-workflows se ejecutan mediante Execute Workflow con la opción Wait For Sub-Workflow Completion activa, permitiendo que el Manager espere la respuesta antes de continuar.

## Trazabilidad

Cada ejecución queda registrada en Google Sheets con información sobre:

- Worker invocado
- Parámetros enviados
- Respuesta devuelta
- Estado de la ejecución

## Archivos

- `manager_modulo2_opisacco_ezequiel.json`
- `worker1_modulo2_opisacco_ezequiel.json`
- `worker2_modulo2_opisacco_ezequiel.json`
- `preentrega_modulo2_opisacco_ezequiel.pdf`

## Proyecto

Asistente para la Gestión de Reclamos de Calidad.
