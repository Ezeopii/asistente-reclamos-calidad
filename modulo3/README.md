# Módulo 3 - Memoria Persistente y Contexto

En este módulo continué el proyecto del **Asistente para la Gestión de Reclamos de Calidad** desarrollado en los módulos anteriores.

Partiendo de la arquitectura multi-agente del Módulo 2, incorporé una capa de memoria persistente para que el sistema pueda recuperar contexto entre ejecuciones y evitar perder información relevante de una sesión.

El objetivo principal fue combinar memoria de corto y largo plazo utilizando `Session_ID` como identificador único de cada conversación.

---

## Arquitectura

El flujo utiliza dos niveles de memoria:

- **Simple Memory:** conserva el historial reciente de la conversación.
- **Airtable:** funciona como memoria persistente de largo plazo vinculada a cada `Session_ID`.

Antes de ejecutar el Manager, el workflow consulta Airtable para verificar si existe información previa asociada a la sesión.

El circuito funciona de la siguiente manera:

```text
Chat Trigger
↓
Capturar Session_ID
↓
Buscar memoria en Airtable
↓
¿Existe memoria?
├── Sí → Recuperar contexto existente
└── No → Crear memoria inicial
↓
Contexto Normalizado
↓
Manager
↓
Workers / Revisión Humana
↓
Preparar Persistencia
↓
¿Superó 5 intercambios?
├── No → Actualizar contador y estado
└── Sí → Recuperar historial → Resumir contexto → Actualizar memoria consolidada
```

---

## Identificación de sesión

Cada conversación se vincula mediante un identificador único:

`Session_ID`

Este valor se utiliza para buscar, crear y actualizar siempre el registro correspondiente a la misma sesión.

De esta forma se evita mezclar información entre conversaciones diferentes.

La lógica aplicada es:

```text
1 Session_ID = 1 registro de memoria
```

Durante las pruebas se validó que, una vez creado el registro, las ejecuciones siguientes actualicen la misma fila en Airtable en lugar de generar registros nuevos.

---

## Memoria de largo plazo

La base de Airtable utilizada para este módulo se llama:

`Memoria Reclamos Calidad`

La tabla principal es:

`Memoria_Sesiones`

Contiene los siguientes campos:

- `Session_ID`
- `Nombre_Usuario`
- `Fecha_Actualizacion`
- `Resumen_Consolidado`
- `Estado_Caso`
- `Datos_Clave`
- `Contador_Mensajes`

La memoria recuperada desde Airtable se incorpora al contexto del Manager utilizando los delimitadores:

```text
[INICIO DE CONTEXTO COMPARTIDO]

...

[FIN DEL CONTEXTO COMPARTIDO]
```

El contenido incluido dentro de este bloque se utiliza únicamente como información histórica de la sesión y no como instrucciones para el agente.

---

## Memoria híbrida

La solución combina dos tipos de memoria.

### Memoria de corto plazo

Se utiliza `Simple Memory` para conservar los intercambios recientes de la conversación.

Esto permite que el agente mantenga continuidad dentro de una misma sesión.

### Memoria de largo plazo

Airtable conserva únicamente información resumida y necesaria para futuras ejecuciones.

De esta forma se evita almacenar conversaciones completas y consumir innecesariamente la ventana de contexto del modelo.

---

## Summarization

El workflow mantiene un contador de mensajes por sesión.

Cuando la conversación supera los **5 intercambios**, se activa una rama específica de resumen.

El proceso es:

```text
Contador_Mensajes > 5
↓
Recuperar Historial Corto
↓
Modelo de resumen
↓
JSON estructurado
↓
Actualizar Airtable
↓
Contador_Mensajes = 0
```

Durante las pruebas, el nodo `Recuperar Historial Corto` recuperó correctamente los mensajes correspondientes a la misma sesión antes de ejecutar la consolidación.

Luego de guardar la nueva memoria, el contador vuelve a `0` para comenzar un nuevo ciclo.

---

## Formato de salida del resumen

El modelo encargado de consolidar la memoria debe devolver exclusivamente la siguiente estructura:

```json
{
  "asunto_principal": "",
  "puntos_clave": [],
  "accion_requerida": ""
}
```

Esto permite guardar únicamente la información necesaria para continuar la gestión del reclamo.

---

## Contrato de datos - JSON Schema

Como mejora respecto de los módulos anteriores, incorporé un **JSON Schema** para documentar formalmente el contrato de salida del proceso de consolidación de memoria.

El esquema establece como obligatorios los campos:

- `asunto_principal`
- `puntos_clave`
- `accion_requerida`

También restringe la incorporación de propiedades adicionales no contempladas.

El archivo utilizado es:

`esquema_memoria_modulo3.json`

El objetivo es que la estructura esperada no dependa únicamente de lo indicado en el prompt, sino que exista un contrato de datos explícito que pueda utilizarse posteriormente para realizar validaciones automáticas.

En esta versión, el JSON Schema funciona como documentación formal del contrato de salida. Su validación automática dentro del workflow queda contemplada como una mejora futura.

---

## Persistencia y actualización

La actualización de memoria se realiza utilizando `Session_ID` como referencia.

Cuando la sesión ya existe:

- se recupera el registro anterior;
- no se crea una nueva fila;
- se incrementa el contador de mensajes;
- se actualiza la fecha;
- se mantiene el estado del caso;
- cuando corresponde, se reemplaza el resumen consolidado.

Esto permite trabajar siempre sobre una única memoria vigente por sesión.

Cuando el proceso de resumen finaliza correctamente, el contador vuelve a `0` y comienza un nuevo ciclo de acumulación de contexto.

---

# Gestión de errores técnicos y resiliencia

Además de los casos funcionales del workflow, se identificaron los principales puntos donde podría producirse una falla técnica.

La versión actual diferencia entre situaciones controladas por la lógica del flujo y errores externos que requieren revisión.

## Sesión inexistente en Airtable

Si la búsqueda por `Session_ID` no encuentra un registro, esta situación no se considera un error técnico.

El nodo `IF` deriva automáticamente la ejecución hacia la rama de sesión nueva:

```text
Crear Memoria Inicial
```

Luego de crear el registro, el workflow continúa normalmente hacia el Manager.

De esta manera, la ausencia de información previa no interrumpe la ejecución.

---

## Solicitud no clasificada

Si el Manager recibe una solicitud que no puede asignar a ninguna de las categorías disponibles, devuelve:

```text
NO_CLASIFICADO
```

El flujo deriva entonces el caso hacia:

```text
Revision Humana
```

Esto evita forzar una clasificación o ejecutar automáticamente un Worker incorrecto.

---

## Prevención de duplicados

Uno de los riesgos identificados durante las pruebas fue la posibilidad de generar más de un registro para una misma sesión.

Para evitarlo, tanto la búsqueda como las actualizaciones utilizan `Session_ID` como referencia.

Cuando la sesión ya existe, el workflow actualiza el mismo registro de Airtable en lugar de crear una nueva fila.

Durante las pruebas se validó este comportamiento verificando que el contador de mensajes se actualizara sobre el mismo registro.

---

## Respuesta incorrecta del modelo de resumen

Existe la posibilidad de que el modelo devuelva:

- texto adicional;
- información no solicitada;
- campos inesperados;
- una estructura diferente a la requerida.

Para reducir este riesgo, el System Prompt del resumidor establece explícitamente que:

- no debe inventar información;
- no debe copiar conversaciones completas;
- no debe incluir HTML;
- no debe incluir logs técnicos;
- debe devolver exclusivamente JSON;
- debe utilizar los campos definidos.

Además, se incorporó `esquema_memoria_modulo3.json` para formalizar el contrato esperado.

Actualmente el esquema funciona como documentación del contrato.

Como mejora futura, se contempla realizar una validación automática contra el JSON Schema antes de permitir la escritura del resultado en Airtable.

---

## Error de conexión con Airtable

Puede ocurrir que Airtable no responda, que exista un problema temporal de conectividad o que una credencial deje de ser válida.

En la versión actual del workflow no existe una ruta alternativa automática para este escenario.

Si ocurre una falla de lectura o escritura en Airtable, la ejecución debe considerarse incompleta y revisarse desde el historial de ejecuciones de n8n antes de continuar.

No se debe asumir que la memoria fue correctamente actualizada hasta confirmar que el nodo finalizó de forma exitosa.

Como mejora futura se plantea incorporar:

- reintentos automáticos ante fallas temporales;
- una ruta específica para errores de Airtable;
- alerta al equipo de Calidad;
- registro del error sin modificar la memoria anterior.

---

## Error del proveedor del modelo

También puede producirse una falla en el modelo utilizado por el Manager o por el proceso de summarization.

En la versión actual, si el proveedor del modelo no responde correctamente, la ejecución requiere revisión.

Como mejora futura se contempla incorporar un mecanismo de fallback:

```text
Modelo principal
↓
¿Ejecución correcta?
├── Sí → Continuar
└── No → Utilizar modelo alternativo
```

Esto permitiría aumentar la disponibilidad del flujo ante fallas externas del proveedor principal.

---

## Falla durante la actualización final de memoria

Otro escenario identificado es que el análisis y el resumen se ejecuten correctamente, pero falle la escritura final en Airtable.

En este caso, la memoria no debe considerarse actualizada hasta confirmar que el nodo de persistencia finalizó correctamente.

Como mejora futura se contempla agregar una validación posterior a la escritura para verificar:

- que el `Session_ID` siga siendo el mismo;
- que el registro haya sido actualizado;
- que el contador tenga el valor esperado;
- que el nuevo resumen se encuentre disponible.

---

## Mejoras de resiliencia identificadas

A partir de las pruebas realizadas y del análisis de los posibles puntos de falla, quedaron identificadas las siguientes mejoras para futuras versiones:

- reintentos automáticos ante fallas temporales;
- rutas específicas para errores técnicos;
- fallback hacia un segundo modelo;
- validación automática utilizando JSON Schema;
- alertas cuando falle la persistencia;
- registro diferenciado de errores técnicos;
- verificación posterior de las escrituras realizadas en Airtable.

Estas mejoras todavía no forman parte de la versión actual del workflow, pero quedan documentadas como próximos pasos para aumentar su resiliencia.

---

## Protección de la memoria

La base persistente no almacena información innecesaria.

No se guardan:

- conversaciones completas;
- payloads completos del Chat Trigger;
- HTML;
- logs técnicos de n8n;
- metadatos de ejecución;
- respuestas completas del agente.

Solo se conservan:

- resumen consolidado;
- estado del caso;
- datos clave;
- contador de mensajes;
- identificación de la sesión.

---

## Pruebas realizadas

Durante las pruebas se verificó:

- creación de memoria para una sesión nueva;
- recuperación de memoria para una sesión existente;
- aislamiento correcto por `Session_ID`;
- actualización sobre el mismo registro;
- prevención de duplicados;
- incremento del contador;
- recuperación del historial corto;
- activación de summarization al superar 5 intercambios;
- generación de salida JSON;
- actualización del resumen consolidado;
- reinicio del contador;
- reutilización del contexto en ejecuciones posteriores;
- derivación a revisión humana cuando el Manager devuelve `NO_CLASIFICADO`.

---

## Archivos

La carpeta del Módulo 3 contiene:

```text
modulo3/
├── README.md
├── manager_modulo3_opisacco_ezequiel.json
├── esquema_memoria_modulo3.json

```

Los Workers utilizados por el Manager continúan siendo los desarrollados en el Módulo 2.

---

## Proyecto

**Asistente para la Gestión de Reclamos de Calidad**

El proyecto continúa evolucionando módulo a módulo, manteniendo la arquitectura desarrollada anteriormente e incorporando nuevas capacidades sobre el mismo flujo.
