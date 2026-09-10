Módulo 8 - QA Automatizado con AI-as-a-Judge

Último checkpoint del proyecto Asistente para la Gestión de Reclamos de Calidad.

Objetivo

Agregar una capa de supervisión sobre las respuestas del agente principal para evaluar su exactitud antes de habilitar una salida real.

El supervisor recibe:

query_original

contexto_airtable

respuesta_propuesta

y devuelve mediante Structured Output:

{
  "exactitud_factual": 5,
  "estado_veredicto": "ACEPTADO"
}

Estados válidos:

ACEPTADO

CORREGIR

RECHAZADO

Arquitectura del supervisor

Agente
  ↓
Preparar Payload QA
  ↓
AI Judge
  ↓
Structured Output
  ↓
Registrar QA - Airtable
  ↓
Switch - Veredicto QA
  ├── ACEPTADO  → Gmail
  ├── CORREGIR  → Crítica → 1 reintento
  └── RECHAZADO → Slack → Wait / revisión humana

Rúbrica

5 = correcta, completa y respaldada
4 = correcta con omisión menor
3 = parcialmente correcta / requiere corrección
2 = no resuelve un dato disponible o contiene error importante
1 = inventa o contradice el contexto

5-4 → ACEPTADO
3   → CORREGIR
2-1 → RECHAZADO

Si la fuente de referencia indica que un dato no está disponible, una respuesta como No sé se considera correcta siempre que no agregue información inventada.

Auditoría

Los resultados se registran en Airtable (QA_Log) antes del Switch.

Campos principales:

Arquitectura

Query

Contexto_Airtable

Respuesta_Propuesta

Exactitud_Factual

Estado_Veredicto

Intento

Categoria_Error

Created Time

Test A/B

Se compararon exactamente 10 corridas:

5 con Arquitectura A - Sin RAG

5 con Arquitectura B - Con RAG

Resultados

Prueba

A - Sin RAG

B - Con RAG

Hora de planificación

3/5 - CORREGIR

5/5 - ACEPTADO

Rechazo de mercadería

2/5 - RECHAZADO

5/5 - ACEPTADO

Retiro de devolución

2/5 - RECHAZADO

5/5 - ACEPTADO

Servicio aéreo estándar

2/5 - RECHAZADO

5/5 - ACEPTADO

Garantía no documentada

5/5 - ACEPTADO

5/5 - ACEPTADO

Precisión media normalizada:

Arquitectura A: 56%
Arquitectura B: 100%

Costo

Durante el checkpoint no se persistieron tokens/precio monetario por ejecución, por lo que no se inventó un costo USD.

Se utilizó como proxy auditable la cantidad de invocaciones:

A: 2 llamadas LLM por corrida
B: 3 llamadas LLM + retrieval por corrida

Proyección a 1.000 ejecuciones:

A: 2.000 llamadas LLM
B: 3.000 llamadas LLM + 1.000 retrievals

Recomendación

Para consultas documentales y casos de Calidad se recomienda la Arquitectura B con RAG, ya que obtuvo 100% de precisión normalizada en la muestra frente a 56% de A.

La Arquitectura A puede mantenerse para clasificación o tareas de bajo riesgo que no dependan de documentación.
