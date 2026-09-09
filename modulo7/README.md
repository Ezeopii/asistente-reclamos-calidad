Módulo 7 - Diseño Arquitectónico de un Sistema Agéntico Vertical

En este módulo continué el desarrollo del Asistente para la Gestión de Reclamos de Calidad, aplicando la arquitectura construida en los módulos anteriores a una vertical real de negocio.

Vertical seleccionada

Logística - Gestión de Reclamos de Calidad

El proceso elegido corresponde a la recepción, análisis y seguimiento de reclamos vinculados a operaciones logísticas.

Problema relevado

Actualmente, cuando llega un reclamo, primero se debe realizar un relevamiento para:

entender qué ocurrió;

identificar al cliente y la operación;

revisar correos y antecedentes;

controlar la documentación disponible;

detectar información faltante;

preparar el seguimiento correspondiente.

El análisis inicial es realizado principalmente por el equipo de Calidad y Procesos, con participación de Operaciones, Administración u otras áreas según el caso.

Como baseline se tomó un tiempo aproximado de:

30 a 45 minutos por reclamo

Framework de priorización

Se evaluaron tres criterios principales:

Impacto operativo: 5/5
Viabilidad no-code: 4/5
Adopción esperada: 4/5

El proceso presenta una prioridad alta para automatización porque combina impacto operativo, viabilidad técnica y posibilidad de mantener revisión humana para las decisiones sensibles.

Arquitectura multi-agente

La solución propuesta utiliza una red de agentes especializados:

Manager
├── Analizador de Reclamos
├── Agente RAG - Consulta Documental
└── Redactor de Comunicaciones
        ↓
Revisión Humana

Manager

Recibe la solicitud y decide qué agente debe intervenir.

No puede tomar decisiones sobre:

responsabilidad;

pagos;

indemnizaciones;

cobertura de seguro;

cierre definitivo del reclamo.

Analizador de Reclamos

Su función es revisar el caso e identificar:

cliente y operación;

incidente;

antecedentes;

documentación disponible;

datos faltantes;

nivel de riesgo.

Agente RAG - Consulta Documental

Consulta documentación vigente relacionada con el caso.

Las fuentes previstas incluyen:

POE - Procedimientos Operativos Estándar

IO - Instrucciones Operativas

contratos;

cartas de oferta;

correos y antecedentes;

denuncias;

documentación adjunta al reclamo;

otros respaldos disponibles en los sistemas.

Si encuentra información contradictoria o no puede confirmar cuál es la versión vigente, debe escalar el caso a revisión humana.

Redactor de Comunicaciones

Prepara borradores utilizando el análisis del reclamo y la información documental disponible.

Las comunicaciones sensibles requieren revisión humana antes del envío.

Principio de menor privilegio

Cada agente accede solamente a las herramientas necesarias para realizar su tarea.

Ejemplo:

Manager    → lectura de contexto
Analizador → Gmail, Airtable y HubSpot en lectura
RAG        → documentación en lectura
Redactor   → contexto + creación de borradores
Humano     → aprobación y envío

Esto evita entregar permisos innecesarios a cada componente del sistema.

Context Engineering

Cada agente recibe solamente la información necesaria para su función.

Manager

mensaje actual;

remitente;

memoria breve del caso.

Analizador

reclamo recibido;

antecedentes;

cliente;

operación;

documentación disponible.

RAG

consulta;

cliente u operación relacionada;

documentación vigente aplicable.

Redactor

análisis del reclamo;

faltantes;

antecedentes necesarios;

información documental recuperada.

Propuesta de valor

La solución busca reducir el tiempo administrativo dedicado al relevamiento y seguimiento inicial de reclamos.

El objetivo es automatizar tareas repetitivas para que el equipo de Calidad pueda dedicar más tiempo a los casos que realmente requieren análisis y criterio humano.

KPIs iniciales

Se definieron los siguientes indicadores como objetivos de referencia:

Tiempo actual de análisis: 30-45 min
Reducción del tiempo administrativo: >= 30%
Reclamos clasificados correctamente: >= 90%
Consultas documentales con respaldo vigente: >= 85%
Borradores con pocas correcciones: >= 80%
Casos críticos escalados correctamente: 100%

Estos valores funcionan como objetivos iniciales y deberían ajustarse una vez que el sistema se utilice con datos reales de operación.

Semáforo de riesgo

Verde - Autónomo

El sistema puede:

clasificar solicitudes;

identificar cliente y operación;

consultar documentación;

detectar información faltante;

actualizar memoria y logs;

preparar borradores.

Amarillo - Revisión humana

Requieren revisión:

respuestas al cliente;

pedidos de documentación;

cambios importantes de estado;

recomendaciones sobre seguro;

posibles responsabilidades;

situaciones no claramente definidas en la documentación.

Rojo - Siempre humano

El sistema no puede decidir de forma autónoma:

aceptación de responsabilidad;

responsable de una pérdida;

indemnizaciones;

pagos o reintegros;

cobertura de seguro;

cierre definitivo;

decisiones legales o comerciales.

Ante estos casos, la acción debe pausarse y escalarse a una persona autorizada mediante Slack o Gmail.

Scorecard

Se diseñó un scorecard simple de 0 o 1 punto por criterio.

1 = información suficiente para avanzar
0 = información faltante o no confirmada

Criterios evaluados:

Cliente y operación

Incidente

Documentación

Antecedentes

Respaldo documental

Valor de mercadería

Datos faltantes

Riesgo del caso

Resultado

7-8 → Caso completo
4-6 → Parcialmente completo
0-3 → Información insuficiente

El puntaje sirve para medir qué tan completo está el caso.

No define responsabilidad ni autoriza decisiones económicas.

Si existe impacto económico, riesgo legal, posible responsabilidad o información contradictoria, el caso debe escalarse sin importar el puntaje obtenido.
