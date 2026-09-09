# Módulo 5 - Agente con Conocimiento Organizacional (RAG)

En este módulo continué el desarrollo del **Asistente para la Gestión de Reclamos de Calidad**, partiendo del workflow del Módulo 4 e incorporando una base de conocimiento documental mediante una arquitectura RAG.

El objetivo fue lograr que el agente pueda consultar documentación operativa real antes de responder, utilizando información recuperada desde un procedimiento interno y evitando inventar respuestas cuando el dato no se encuentra disponible.

---

## Documento utilizado

Se utilizó como documento maestro un **POE - Procedimiento Operativo Estándar** correspondiente a la distribución de Boston Scientific.

El documento contiene:

- procedimientos operativos;
- responsabilidades;
- condiciones comerciales;
- plazos de servicio;
- logística inversa;
- facturación;
- seguimiento;
- tablas de servicios;
- medidas y pesos de cargas.

Se eligió este documento porque permite probar tanto recuperación semántica como lectura de títulos, subtítulos, listas y tablas.

---

## Procesamiento del documento

El archivo fue procesado mediante **LlamaCloud / LlamaParse**.

Durante la prueba se verificó que el parser mantuviera correctamente:

- títulos;
- subtítulos;
- listas;
- estructura del procedimiento;
- tablas con filas y columnas.

Como ejemplo, se validaron secciones como:

```text
10 Desarrollo del procedimiento operativo
10.1 Proceso General
10.1.1 Recolección
```

También se verificaron tablas relacionadas con:

- tipo de servicio;
- volúmenes;
- medidas de cajas;
- pesos de cajas;
- hitos de seguimiento.

### Aclaración

La cuenta gratuita utilizada no permitió crear un Data Source persistente dentro de LlamaCloud.

Por este motivo, se utilizó el módulo `Parse` para procesar el documento y validar la correcta interpretación de su estructura.

---

## Arquitectura RAG

La arquitectura agregada al workflow está dividida en dos bloques.

### Carga documental

```text
Upload your file here
↓
Insert Data to Store
├── Default Data Loader
└── Embeddings Google Gemini
```

Este bloque permite:

1. cargar el documento;
2. dividirlo en fragmentos;
3. generar embeddings;
4. almacenar esos fragmentos en el Vector Store.

---

## Modelo de Embeddings

Para generar los embeddings se utilizó:

```text
Google Gemini
models/gemini-embedding-001
```

Se utilizó Gemini porque la cuenta de OpenAI disponible no contaba con créditos para ejecutar embeddings.

La misma configuración de embeddings se utiliza tanto para cargar como para consultar la información.

---

## Vector Store

Se utilizó el Vector Store disponible en n8n mediante memoria interna.

La clave configurada fue:

```text
poe_boston_m5
```

La misma clave se utiliza para:

- insertar documentos;
- recuperar información.

Esto permite que ambos nodos trabajen sobre la misma base documental.

### Limitación

El Vector Store utilizado es de prueba y trabaja en memoria.

Esto significa que la información puede perderse si la instancia de n8n se reinicia.

Como mejora futura, se plantea utilizar una base vectorial persistente.

---

## Query Data Tool

Para recuperar información del POE se configuró:

```text
Name: poe_boston
Operation Mode: Retrieve Documents (As Tool for AI Agent)
Memory Key: poe_boston_m5
Limit: 4
Include Metadata: ON
Rerank Results: OFF
```

La descripción utilizada fue:

```text
Consulta el POE de Boston Scientific para responder preguntas sobre procedimientos operativos,
responsabilidades, plazos, documentación, logística inversa, facturación y condiciones del servicio.
```

---

## Top-K

En la versión de n8n utilizada, el parámetro `Limit` cumple la función práctica de definir cuántos fragmentos se recuperan por consulta.

Se configuró:

```text
Limit = 4
```

Esto equivale a trabajar con un Top-K de 4.

Se eligió este valor porque permite recuperar suficiente contexto sin enviar demasiada información al agente.

Durante las pruebas, cuatro fragmentos fueron suficientes para responder correctamente las consultas realizadas.

---

## Minimum Score

En la versión 2.28.6 de n8n utilizada para esta entrega, el nodo `Query Data Tool` no muestra un campo independiente para configurar `Minimum Score`.

Por este motivo, la calidad de recuperación se controló mediante:

- Limit = 4;
- descripción específica de la herramienta;
- recuperación por embeddings;
- validación manual con cinco preguntas ciegas.

Como mejora futura, se puede utilizar un Vector Store que permita configurar un umbral mínimo de similitud de forma explícita.

---

## Agente RAG

Se agregó un nuevo agente especializado:

```text
Agente RAG - Consulta POE
```

Su función es responder exclusivamente consultas relacionadas con la documentación operativa.

El agente utiliza como herramienta:

```text
Query Data Tool → poe_boston
```

y recibe las consultas derivadas desde el Manager.

---

## Nueva categoría del Manager

Se agregó una nueva categoría a la taxonomía:

```text
CONSULTAR_POE
```

Esta categoría se utiliza cuando la solicitud corresponde a una consulta sobre:

- procedimientos;
- responsabilidades;
- plazos;
- documentación;
- logística inversa;
- facturación;
- condiciones comerciales;
- información incluida en el POE.

El Switch deriva estas consultas al `Agente RAG - Consulta POE`.

La arquitectura queda:

```text
Manager
↓
Enrutar Solicitud
├── ANALIZAR_RECLAMO
├── REDACTAR_COMUNICACION
├── CONSULTAR_POE
└── NO_CLASIFICADO
```

---

## Reglas del System Prompt RAG

El agente documental fue configurado con reglas específicas para evitar respuestas inventadas.

Las reglas principales son:

- consultar obligatoriamente la herramienta `poe_boston`;
- responder solamente con información recuperada del POE;
- no utilizar conocimiento general para completar datos faltantes;
- indicar la sección o fuente del documento cuando sea posible;
- no inventar información.

También se agregó la regla de contingencia:

```text
Si la información solicitada no aparece en los fragmentos recuperados, responder exactamente:

"No sé"
```

Esto permite reducir el riesgo de alucinaciones.

---

## Validación con preguntas ciegas

Se realizaron cinco preguntas utilizando lenguaje simple e informal.

### Pregunta 1

```text
¿A qué hora tiene que llegar la planificación para organizar los retiros?
```

Respuesta:

```text
17:00 hs del mismo día.
```

Resultado:

```text
Correcto
```

---

### Pregunta 2

```text
Si el cliente rechaza la mercadería cuando llega, ¿qué se termina cobrando?
```

Respuesta:

```text
Doble tarifa, contemplando ida, regreso y seguro correspondiente para ambos recorridos.
```

Resultado:

```text
Correcto
```

---

### Pregunta 3

```text
¿Qué información tengo que mandar para pedir que retiren una devolución?
```

Respuesta recuperada:

```text
- Número de RMA
- Código de cliente
- Origen
- Destino
- Cantidad de bultos
- Tipo de devolución
- Motivo de la devolución
```

Resultado:

```text
Correcto
```

---

### Pregunta 4

```text
Si mando algo por avión con el servicio normal, ¿cuánto tarda?
```

Respuesta:

```text
24 horas hábiles (D+1) desde la recolección.
```

Resultado:

```text
Correcto
```

---

### Pregunta 5

```text
¿Qué garantía tienen los dispositivos médicos de Boston Scientific?
```

Respuesta:

```text
No sé
```

Resultado:

```text
Correcto
```

El dato no estaba presente en el POE y el agente aplicó correctamente la regla de contingencia.

---

## Resultado de la validación

La prueba obtuvo:

```text
5 respuestas correctas de 5
Precisión: 100%
```

La validación mostró que el agente pudo recuperar información utilizando lenguaje distinto al contenido exacto del documento.

También se comprobó que el agente no inventó una respuesta cuando la información no estaba disponible.

---

## Plan de gobernanza

La base documental debe mantenerse actualizada para evitar respuestas incorrectas.

### Frecuencia

La documentación debe revisarse:

- cada 3 meses;
- cuando cambie un procedimiento;
- cuando cambien condiciones del servicio;
- cuando exista una nueva versión aprobada.

### Responsable

El responsable principal será:

```text
Calidad y Procesos
```

con participación del responsable operativo del servicio cuando corresponda.

### Reemplazo de documentos

Cuando exista una nueva versión aprobada:

- se debe actualizar el documento;
- se debe retirar la versión anterior;
- se debe volver a ejecutar la validación.

### Eliminación

Un documento debe eliminarse cuando:

- deje de aplicar al servicio;
- haya sido reemplazado;
- contenga información desactualizada.

### Validación posterior

Después de cada cambio se deben realizar nuevamente preguntas de prueba para confirmar que:

- la información correcta sigue siendo recuperada;
- no se generan respuestas contradictorias;
- la regla `No sé` continúa funcionando.

---

## Evolución del proyecto

El proyecto mantiene la arquitectura construida en los módulos anteriores.

```text
Módulo 1
Agente inicial
↓
Módulo 2
Arquitectura Manager - Worker
↓
Módulo 3
Memoria persistente
↓
Módulo 4
Gmail + HubSpot + Slack
↓
Módulo 5
RAG + Base Documental
```

De esta manera, cada módulo amplía el mismo sistema sin comenzar nuevamente desde cero.

---

## Archivo del módulo

El workflow debe exportarse desde n8n con el nombre:

```text
checkpoint5_ezequiel_opisacco.json
```

La carpeta de GitHub queda:

```text
modulo5/
├── README.md
└── checkpoint5_ezequiel_opisacco.json
```

El PDF de la pre-entrega se entrega por separado y no se incluye dentro del repositorio.

---

## Proyecto

**Asistente para la Gestión de Reclamos de Calidad**

Repositorio general:

https://github.com/Ezeopii/asistente-reclamos-calidad
