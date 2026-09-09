
# Módulo 6 - Voice AI: STT + AI Agent + TTS

En este módulo continué el desarrollo del **Asistente para la Gestión de Reclamos de Calidad**, partiendo del workflow del Módulo 5 e incorporando una capa de voz de extremo a extremo.

El objetivo fue permitir que una persona pueda enviar una nota de voz por Telegram, transcribirla, procesarla mediante un agente de IA y recibir una respuesta nuevamente en formato de audio.

---

## Arquitectura implementada

El circuito de voz quedó compuesto por los siguientes pasos:

```text
Webhook
↓
Telegram - Get a file
↓
Code - Renombrar Audio OGG
↓
HTTP Request - Groq Whisper
↓
IF - Transcripción válida?
├── FALSE → STOP - Audio inválido
└── TRUE
     ↓
AI Agent - Respuesta Voz
     ↓
ElevenLabs - Generar Respuesta de Voz
     ↓
Telegram - Enviar Respuesta de Audio
     ↓
Compliance - Eliminar Binarios
```

---

## Entrada de voz

La entrada se realizó mediante un bot de Telegram conectado a un Webhook de n8n.

La nota de voz enviada por Telegram se descarga utilizando:

```text
Telegram - Get a file
```

El archivo se obtiene como dato binario bajo la propiedad:

```text
data
```

Este nombre se mantiene durante el procesamiento para que pueda ser utilizado por los nodos de transcripción.

---

## Webhook

El Webhook se configuró para recibir los mensajes enviados desde Telegram.

Configuración principal:

```text
HTTP Method: POST
Path: telegram-voz-m6
Authentication: None
Respond: Immediately
```

Como n8n se ejecutó de forma local, se utilizó un túnel público mediante ngrok para permitir que Telegram pudiera acceder al Webhook.

---

## Normalización del archivo de audio

Telegram entregaba el archivo de voz con extensión:

```text
.oga
```

La API utilizada para la prueba esperaba una extensión compatible como:

```text
.ogg
```

Por este motivo se agregó un paso intermedio para ajustar únicamente la metadata del archivo:

```text
Code - Renombrar Audio OGG
```

El archivo quedó normalizado como:

```text
File Name: audio.ogg
File Extension: ogg
Mime Type: audio/ogg
```

---

## OpenAI Whisper

La consigna requería configurar el nodo nativo:

```text
OpenAI - Transcribe a Recording
```

Se configuraron los siguientes parámetros:

```text
Resource: Audio
Operation: Transcribe a Recording
Input Data Field Name: data
Language of the Audio File: es
```

Esto cumple con la parametrización solicitada para el nodo Whisper.

### Limitación encontrada

La cuenta de OpenAI utilizada no contaba con créditos disponibles para ejecutar la API.

Por este motivo, el nodo quedó correctamente configurado, pero la prueba funcional de transcripción se realizó utilizando Whisper mediante Groq.

---

## Transcripción funcional con Groq

Para poder completar el circuito sin cargar saldo en OpenAI, se utilizó:

```text
Groq
whisper-large-v3-turbo
```

La llamada se realizó mediante un nodo:

```text
HTTP Request
```

Configuración principal:

```text
POST
https://api.groq.com/openai/v1/audio/transcriptions
```

Body:

```text
file → data
model → whisper-large-v3-turbo
language → es
response_format → json
```

La transcripción devolvió correctamente texto en español.

Ejemplo:

```text
Necesito ayuda con un reclamo de calidad.
```

---

## Ruta de contingencia

Después de la transcripción se agregó:

```text
IF - Transcripción válida?
```

La condición utilizada fue:

```text
{{ ($json.text || '').trim() }}
```

Operador:

```text
is not empty
```

La lógica queda:

```text
TRUE → continúa al agente
FALSE → STOP - Audio inválido
```

Esto permite cortar de forma segura el flujo cuando la transcripción llega vacía o no contiene información válida.

---

## AI Agent

El texto transcripto se envía al nodo:

```text
AI Agent - Respuesta Voz
```

La entrada utilizada es:

```text
{{ $('HTTP Request').first().json.text }}
```

El agente utiliza un modelo de chat conectado mediante OpenRouter.

---

## Contención financiera

Dentro del System Message se agregó una regla obligatoria:

```text
IMPORTANTE: todas tus respuestas deben tener un máximo estricto de 200 caracteres.

Nunca superes los 200 caracteres.
```

El objetivo es reducir la cantidad de caracteres enviados posteriormente a la API de Text to Speech y evitar respuestas demasiado largas para una interfaz de voz.

También ayuda a que la conversación sea más rápida y simple para el usuario.

---

## Text to Speech con ElevenLabs

Para convertir la respuesta del agente nuevamente en audio se utilizó:

```text
ElevenLabs - Generar Respuesta de Voz
```

Configuración:

```text
Resource: Speech
Operation: Text to Speech
Model: eleven_multilingual_v2
```

El texto recibido por ElevenLabs es:

```text
{{ $('AI Agent - Respuesta Voz').first().json.output }}
```

El nodo genera un archivo MP3 binario bajo:

```text
data
```

---

## Configuración de voz

Se utilizó el modelo:

```text
Eleven Multilingual v2
```

Para una voz institucional se buscó mantener:

- estabilidad media;
- similitud alta;
- velocidad normal;
- baja exageración de estilo.

La versión del nodo nativo de ElevenLabs utilizada en n8n no expone directamente los sliders de Stability y Similarity.

Por este motivo, la configuración de estos parámetros se documentó desde la interfaz web de ElevenLabs.

---

## Salida por Telegram

La respuesta de audio se envía nuevamente al mismo chat mediante:

```text
Telegram - Enviar Respuesta de Audio
```

Configuración:

```text
Resource: Message
Operation: Send Audio
Binary File: ON
Input Binary Field: data
```

Chat ID:

```text
{{ $('Webhook').first().json.body.message.chat.id }}
```

Esto permite que el usuario reciba directamente la respuesta generada por ElevenLabs en el mismo chat donde envió la nota de voz.

---

## Compliance y privacidad

Al finalizar el envío del audio se ejecuta:

```text
Compliance - Eliminar Binarios
```

El nodo deja como salida únicamente:

```text
estado = Audio procesado y eliminado de la ejecución activa
```

y no continúa transportando el archivo binario hacia pasos posteriores.

Además:

- el audio no se almacena en Airtable;
- el audio no se guarda en HubSpot;
- no se incorpora a la base documental RAG;
- no se utiliza como memoria persistente.

Para un entorno productivo también sería necesario configurar una política de retención mínima de ejecuciones dentro de n8n.

---

## Prueba end-to-end

Se realizó una prueba completa:

```text
Telegram
↓
Webhook
↓
Descarga del audio
↓
Whisper STT
↓
Validación
↓
AI Agent
↓
ElevenLabs
↓
Telegram
↓
Compliance
```

El resultado fue exitoso:

- la nota de voz fue recibida;
- el audio fue transcripto;
- el agente generó una respuesta;
- ElevenLabs generó el MP3;
- Telegram devolvió correctamente el audio;
- el flujo finalizó sin mantener el binario.

---

## Evolución del proyecto

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
↓
Módulo 6
Voice AI + STT + TTS
```

El proyecto sigue creciendo sobre la misma arquitectura y no se reinicia en cada módulo.

---

## Archivo del módulo

El workflow se exporta desde n8n con el nombre:

```text
checkpoint6_ezequiel_opisacco.json
```

La carpeta de GitHub queda:

```text
modulo6/
├── README.md
└── checkpoint6_ezequiel_opisacco.json
```

El PDF de la pre-entrega se entrega por separado y no se incluye dentro del repositorio.

---

## Proyecto

**Asistente para la Gestión de Reclamos de Calidad**

Repositorio general:

https://github.com/Ezeopii/asistente-reclamos-calidad
