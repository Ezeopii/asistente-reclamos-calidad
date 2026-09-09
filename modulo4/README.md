# Módulo 4 - Integraciones Avanzadas e Interconexión de Sistemas

En este módulo continué el proyecto del **Asistente para la Gestión de Reclamos de Calidad** desarrollado en los módulos anteriores.

Partiendo del workflow del Módulo 3, que ya contaba con memoria persistente y contexto por sesión, incorporé integraciones externas reales para conectar el sistema con herramientas utilizadas en un entorno operativo:

- Gmail
- HubSpot
- Slack

El objetivo principal fue permitir que el asistente reciba reclamos desde una casilla de correo, procese la información, consulte o actualice contactos en un CRM, prepare respuestas para revisión humana y notifique al equipo interno mediante Slack.

---

## Arquitectura general

El flujo principal quedó estructurado de la siguiente manera:

```text
Gmail Trigger
↓
IF - Correo Automatico?
├── TRUE → STOP - Correo Automatico
└── FALSE
        ↓
Capturar Sesion
↓
Memoria persistente / Contexto M3
↓
Manager - Clasificar Solicitud
↓
HubSpot - Buscar Contacto
↓
IF - Existe Contacto?
├── TRUE → HubSpot - Actualizar Contacto
└── FALSE → HubSpot - Crear Contacto
        ↓
Gmail - Crear Borrador HITL
↓
Preparar Slack
↓
Slack - Avisar a Calidad
```

La arquitectura mantiene la memoria y lógica multi-agente desarrollada en módulos anteriores y suma una capa de integración con sistemas externos.

---

## Gmail como canal de entrada

El workflow utiliza Gmail como punto de entrada para los reclamos.

### Configuración

- **Trigger:** Gmail
- **Evento:** `Message Received`
- **Autenticación:** OAuth2
- **Frecuencia de consulta:** cada minuto

El mensaje recibido aporta información como:

- remitente;
- asunto;
- contenido;
- `threadId`;
- identificador del mensaje.

El `threadId` de Gmail se utiliza como `Session_ID`, permitiendo que distintos correos pertenecientes al mismo hilo compartan la misma memoria persistente.

La lógica aplicada es:

```text
1 hilo de Gmail = 1 Session_ID = 1 memoria del reclamo
```

---

## Prevención de bucles de auto-respuesta

Inmediatamente después del Gmail Trigger se incorporó un nodo:

`IF - Correo Automatico?`

Su función es identificar correos automáticos antes de enviarlos al agente.

Las condiciones se evalúan mediante lógica `OR`.

Se controlan los siguientes casos:

- asunto contiene `auto-reply`;
- asunto contiene `out of office`;
- asunto contiene `undeliverable`;
- remitente contiene `no-reply@`.

Si alguna condición se cumple, la ejecución se deriva hacia:

`STOP - Correo Automatico`

y el flujo finaliza sin procesar el mensaje.

Esto evita posibles ciclos infinitos de respuestas automáticas.

---

## Adaptación de la memoria del Módulo 3

El flujo conserva la arquitectura de memoria híbrida desarrollada anteriormente.

La principal modificación fue reemplazar el identificador proveniente del Chat Trigger por el `threadId` de Gmail.

El nodo `Capturar Sesion` normaliza:

- `session_id`
- `mensaje_actual`
- `nombre_usuario`

El contenido del correo se transforma en un mensaje estructurado con:

```text
Asunto
Remitente
Contenido
```

Luego continúa hacia la memoria persistente de Airtable y el Manager.

---

## Integración con HubSpot

HubSpot se utiliza como CRM para consultar y mantener la información de los contactos asociados a los correos entrantes.

La arquitectura implementada utiliza una búsqueda previa antes de realizar cualquier escritura:

```text
HubSpot - Buscar Contacto
↓
IF - Existe Contacto?
├── TRUE → Actualizar contacto existente
└── FALSE → Crear contacto nuevo
```

### Búsqueda de contactos

- **Resource:** Contact
- **Operation:** Search
- **Filtro:** Email
- **Limit:** 1

El correo utilizado para la búsqueda se obtiene del remitente de Gmail.

Ejemplo:

```text
Ezequiel Opisacco <correo@ejemplo.com>
```

se normaliza como:

```text
correo@ejemplo.com
```

De esta forma, el email funciona como referencia para determinar si el contacto ya existe.

---

## Prevención de duplicados en CRM

Antes de realizar la escritura se ejecuta una búsqueda determinista por correo electrónico.

Luego un nodo `IF` verifica si HubSpot devolvió un identificador de contacto.

Esto permite distinguir entre:

```text
Contacto existente
→ actualización
```

y:

```text
Contacto inexistente
→ creación
```

Esta lógica evita crear contactos duplicados y reduce el riesgo de errores asociados a escrituras repetidas.

### Particularidad del conector utilizado

En la versión de n8n utilizada para la entrega, la operación disponible para escritura en contactos es:

`Create or Update a contact`

Por este motivo, ambas ramas utilizan la misma operación técnica.

Sin embargo, la decisión de negocio se realiza previamente mediante:

```text
Search contacts
↓
IF - Existe Contacto?
```

De esta manera, el flujo determina explícitamente si el contacto existe antes de ejecutar la escritura.

---

## Gmail - Human in the Loop

Después de procesar el reclamo y actualizar el CRM, el sistema prepara una respuesta mediante Gmail.

El nodo utilizado es:

- **Resource:** Draft
- **Operation:** Create

El workflow **no envía automáticamente el correo**.

En su lugar, genera un borrador en Gmail para que un integrante del equipo de Calidad pueda:

1. revisar el contenido;
2. realizar modificaciones si fueran necesarias;
3. decidir manualmente si corresponde enviarlo.

La arquitectura queda:

```text
IA / procesamiento
↓
Create Draft
↓
Revisión humana
↓
Envío manual
```

Esto establece una barrera obligatoria de **Human-in-the-loop (HITL)** antes de cualquier comunicación externa definitiva.

---

## Limpieza de payload antes de Slack

Antes de enviar información hacia Slack se incorporó el nodo:

`Preparar Slack`

El nodo utiliza `Manual Mapping` y mantiene desactivada la opción de conservar campos de entrada adicionales.

De esta manera, el payload completo proveniente de Gmail, HubSpot u otros nodos no se transmite al canal.

Los campos enviados se reducen a:

- `cliente`
- `asunto`
- `referencia`
- `clasificacion`
- `estado`
- `mensaje_slack`

No se envían:

- headers completos;
- labels de Gmail;
- objetos completos de HubSpot;
- metadata innecesaria;
- payloads pesados;
- archivos binarios.

Esto permite reducir el tamaño de los mensajes y evitar saturar el canal.

---

## Integración con Slack

Slack se utiliza como canal interno para notificar al equipo de Calidad cuando un reclamo fue procesado.

### Configuración

- **Resource:** Message
- **Operation:** Send
- **Destino:** Channel
- **Tipo:** Simple Text Message
- **Autenticación:** OAuth2

La aplicación de Slack fue configurada con permisos mínimos para enviar mensajes al canal correspondiente.

El mensaje enviado contiene únicamente la información previamente limpia por `Preparar Slack`.

Ejemplo:

```text
Nuevo reclamo procesado

Cliente: Ezequiel Opisacco
Asunto: Reclamo M4 prueba 005
Clasificación: ANALIZAR_RECLAMO
Estado: Borrador preparado - Pendiente de revisión humana
```

---

## Principio de mínimo privilegio

Durante la configuración de las integraciones se buscó limitar los permisos a las funciones necesarias para el workflow.

### Gmail

Se utiliza para:

- recibir correos;
- crear borradores.

El flujo no realiza envíos automáticos.

### HubSpot

Se utiliza solamente para:

- buscar contactos;
- crear o actualizar contactos.

No se utilizaron permisos para:

- Deals;
- Companies;
- Marketing;
- Payments;
- otros objetos no necesarios.

### Slack

La aplicación se configuró únicamente para:

- consultar el canal necesario;
- publicar mensajes en el canal autorizado.

---

## Autenticación

Las integraciones utilizadas fueron configuradas de la siguiente manera:

### Gmail

OAuth2.

### Slack

OAuth2 mediante una aplicación creada específicamente para el proyecto.

### HubSpot

Para las pruebas funcionales se utilizó una **Service Key / token de servicio** con permisos limitados a contactos.

Esta configuración permitió validar correctamente:

- búsqueda de contactos;
- control de duplicados;
- creación;
- actualización.

Como mejora futura queda pendiente migrar esta autenticación a OAuth2 para cumplir completamente con el modelo de autorización planteado para entornos productivos.

---

## Controles preventivos implementados

El workflow incorpora cuatro controles principales:

### 1. Prevención de auto-respuestas

```text
Gmail Trigger
↓
IF - Correo Automatico?
```

Evita ciclos infinitos de respuesta.

### 2. Prevención de duplicados en HubSpot

```text
Search Contact
↓
IF - Existe?
```

Evita crear contactos duplicados.

### 3. Human-in-the-loop

```text
Gmail - Create Draft
```

Evita enviar comunicaciones sin revisión humana.

### 4. Limpieza de payload

```text
Preparar Slack
↓
Slack
```

Reduce la información enviada a mensajería y evita payloads innecesariamente pesados.

---

## Pruebas realizadas

Durante las pruebas se verificó:

- recepción real de correos mediante Gmail;
- identificación de correos normales;
- funcionamiento del filtro anti auto-reply;
- creación de una nueva sesión mediante `threadId`;
- recuperación de memoria del Módulo 3;
- clasificación del reclamo por el Manager;
- búsqueda de contactos por email en HubSpot;
- creación de contacto inexistente;
- actualización de contacto existente;
- creación de borrador en Gmail;
- ausencia de envío automático;
- limpieza del payload antes de Slack;
- envío exitoso de la notificación al canal de Slack.

---

## Seguridad y resiliencia

El diseño incorpora controles preventivos antes de realizar acciones externas.

Se evita depender únicamente de decisiones probabilísticas del agente para:

- detectar auto-respuestas;
- decidir si un contacto existe;
- enviar comunicaciones;
- determinar qué información se transmite a Slack.

Estas decisiones se apoyan en nodos deterministas como:

- `IF`
- `Search`
- `Edit Fields`

Como mejoras futuras se contempla incorporar:

- reintentos automáticos;
- rutas específicas ante errores de APIs externas;
- fallback de proveedores;
- alertas técnicas;
- validación posterior de escrituras;
- migración completa de HubSpot a OAuth2.

---

## Archivo de entrega

El workflow se exportó desde n8n con el nombre:

```text
checkpoint4_ezequiel_opisacco.json
```

La estructura de la carpeta correspondiente es:

```text
modulo4/
├── README.md
└── checkpoint4_ezequiel_opisacco.json
```

---

## Proyecto

**Asistente para la Gestión de Reclamos de Calidad**

El proyecto continúa evolucionando sobre la misma arquitectura desarrollada en los módulos anteriores, incorporando nuevas capacidades sin comenzar nuevamente desde cero.
