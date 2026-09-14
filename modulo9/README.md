Modulo 9 - Gobernanza, Gestion y Monitoreo Financiero de IA

Proyecto: Asistente para la Gestion de Reclamos de Calidad

Este modulo lleva el workflow a una capa de gobierno empresarial: costos, ROI, trazabilidad, seguridad, privacidad y monitoreo operativo.

Evidencias principales

1. Dashboard operativo

Se creo una interfaz en Airtable sobre AI_Run_Summary, alimentada desde n8n.

KPIs definidos:

Ejecuciones totales.

Tasa de gestion inicial autonoma.

Exactitud factual promedio.

Minutos administrativos ahorrados.

Tasa de escalamiento HITL.

Tasa de errores detectados.

Satisfaccion del usuario (pendiente de instrumentar con encuesta real).

Aclaracion: los 30 minutos usados como referencia corresponden a la gestion inicial de un reclamo. Un reclamo completo puede durar dias o semanas.

2. ROI y plan financiero

Escenario utilizado:

20 reclamos por mes.

30 minutos minimos de gestion inicial por reclamo.

USD 10/h como valor economico de referencia; no es un salario ni una tarifa interna de Robinson.

Escenario base: 20 minutos ahorrados por reclamo, conservando 10 minutos para validacion humana.

El PDF diferencia el ROI sobre costo variable de tokens del ROI empresarial completo.

3. Economia del token

Modelo actual:
openai/gpt-4o-mini via OpenRouter.

Estimacion por gestion inicial:

6.500 tokens de entrada.

700 tokens de salida.

7.200 tokens totales.

El TCO separa:

LLM.

Embeddings / RAG.

n8n.

Airtable.

Gmail / Slack.

Mantenimiento.

Los componentes no medidos no se presentan como gratis: quedan marcados como pendientes de medicion o sin costo incremental medido en el prototipo.

4. Prompt Caching

Se analiza Prompt Caching de Anthropic como estrategia complementaria para bloques grandes y fijos de documentacion.

Ejemplo incluido:

20.000 tokens fijos.

10 reutilizaciones.

Ahorro estimado del 78,5% sobre ese bloque usando los multiplicadores oficiales de cache.

El prototipo actual sigue utilizando RAG.

5. Trazabilidad con traceId

El workflow genera un identificador unico:

M9-<execution.id>

Ejemplo validado:

M9-255-START
M9-255-JUDGE

Ambos registros comparten:

traceId = M9-255

Campos principales de AI_Execution_Log:

traceId

sessionId

timestamp

environment

workflowVersion

eventType

nodeName

toolOutput

systemPromptSnapshot

confidenceScore

inputTokens

outputTokens

estimatedCostUSD

humanReviewRequired

Para cualquier sub-workflow o herramienta nueva, el traceId debe transmitirse explicitamente y devolverse sin modificar.

Seguridad y gobernanza

Se creo una API key de desarrollo separada:

RL-Calidad-M9-DEV

Controles:

Limite de USD 5/mes.

Guardrail dedicado.

Solo GPT-4o mini habilitado.

Entrenamiento/publicacion de datos deshabilitados.

Prompt Injection: Block.

Deteccion selectiva de informacion sensible.

La key creada para M9 se usa como evidencia de separacion de entornos; no fue necesario reemplazar la credencial del workflow durante el checkpoint.

Privacidad

Politica propuesta para produccion:

No permitir endpoints que entrenen con datos corporativos.

Exigir Zero Data Retention en endpoints de produccion.

Mantener separado el logging interno de la retencion del proveedor.

Validar providers/endpoints con Sistemas y Legal.

No guardar razonamiento interno del modelo; registrar solamente eventos, outputs, evidencias, scores y decisiones observables.
