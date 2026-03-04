
# Arquitectura LLM → TTS → Usuario (Baja Latencia)

## Objetivo

Diseño de un sistema de **streaming end-to-end** donde cada etapa comienza a procesar datos sin esperar a que la anterior finalice completamente, minimizando la latencia hasta el primer audio reproducible.

---

## Principio Arquitectónico

La clave es el **streaming continuo**:

- Los tokens del modelo de lenguaje se reciben de forma incremental.
- Se agrupan en fragmentos semánticos.
- Cada fragmento se envía inmediatamente al sistema de síntesis de voz.
- El audio generado se transmite al cliente en tiempo real.

No existe procesamiento por lotes completos.

---

## Pipeline Óptimo

LLM (streaming tokens)  
↓  
Sentence Chunker  
↓  
TTS (streaming)  
↓  
Audio Player (streaming)


---

## Servicios Recomendados

### Modelo de Lenguaje (LLM)

Opciones con soporte de streaming:

- Claude API
- GPT-4o con `stream: true`

Características esperadas:

- Recepción incremental de tokens
- Primer token en ~50–100 ms (dependiendo de red y carga)

---

### Síntesis de Voz (TTS)

Ordenados por latencia al primer byte:

| Servicio                | Latencia primer byte | Calidad        | Streaming |
|--------------------------|----------------------|---------------|-----------|
| ElevenLabs (Turbo v2.5)  | ~300 ms              | Excelente     | Sí (WebSocket / HTTP chunked) |
| Cartesia (Sonic)         | ~150 ms              | Muy buena     | Sí (WebSocket nativo) |
| Deepgram Aura            | ~250 ms              | Buena         | Sí |
| OpenAI (tts-1)           | ~400 ms              | Muy buena     | Sí |
| Azure Neural TTS         | ~200 ms              | Muy buena     | Sí (WebSocket) |

**Recomendación técnica:**

- **Cartesia Sonic** para mínima latencia.
- **ElevenLabs Turbo** para equilibrio entre calidad y latencia.

---

## Estrategia de Chunking (Elemento Crítico)

No se debe:

- Enviar token por token al TTS.
- Esperar a que el texto completo esté generado.

Se recomienda un **sentence chunker** con las siguientes reglas:

Acumular tokens hasta detectar:

- Fin de oración: `. ! ?`
- Pausa natural tras ~15–20 palabras
- Inactividad de ~500 ms sin nuevos tokens

Al cumplirse cualquiera de estas condiciones:

→ Enviar el fragmento inmediatamente al TTS.

Este enfoque equilibra:

- Baja latencia (chunks pequeños)
- Calidad prosódica (chunks con sentido semántico)

---

## Arquitectura Recomendada

Cliente (Browser / App)  
↕ WebSocket  
Backend (Node.js / Python)  
├── LLM Stream (HTTPS SSE)  
├── Sentence Chunker (memoria)  
└── TTS Stream (WebSocket)  
↕  
Audio Chunks (PCM / Opus)


### Responsabilidades del Backend

- Orquestar streams
- No almacenar audio
- No persistir estado
- Actuar como proxy inteligente

Backend completamente stateless.

---

## Detalles Técnicos Clave

### 1. Protocolo de Comunicación

- WebSocket entre cliente y backend.
- Permite envío continuo de audio chunks.
- Reduce overhead frente a polling.

---

### 2. Formato de Audio

Preferible:

- `pcm_16000`
- `opus`

Evitar:

- MP3 (introduce latencia adicional por buffering y decodificación).

---

### 3. Reproducción en Cliente

Web:

- Uso de `AudioWorklet` para reproducción en tiempo real.
- Cola de buffers.

Mobile:

- Sistema de buffer incremental.

El objetivo es reproducir audio conforme llega, sin esperar al archivo completo.

---

## Latencias Esperadas (End-to-End)

| Fase                         | Tiempo estimado |
|-------------------------------|----------------|
| Primer token LLM              | ~100–300 ms    |
| Generación primera oración     | ~500–1000 ms   |
| Primer chunk TTS              | ~150–400 ms    |
| Total hasta primer audio      | ~800–1500 ms    |

---

## Opciones para Latencia Aún Más Baja

### TTS Local

- Piper
- Kokoro TTS (GPU)

Ventaja:
- Eliminación de latencia de red (~50 ms)

---

### LLM Local

- Llama 3
- vLLM
- llama.cpp

Requiere GPU para rendimiento óptimo.

---

### Speculative Decoding

Envío anticipado de fragmentos al TTS antes de completar la oración.

Ventaja:
- Menor latencia percibida.

Desventaja:
- Reducción parcial de prosodia.

---

### Voice Model Preloaded

- Carga previa del modelo de voz.
- Evita cold starts.
- Mejora consistencia en producción.

---

## Stack Recomendado

Configuración equilibrada:

- Claude API (streaming)
- Cartesia Sonic (WebSocket)
- Backend ligero en Node.js o Python (FastAPI + asyncio)
- WebSocket hacia el cliente

Arquitectura optimizada para:

- Baja latencia
- Escalabilidad
- Separación clara de responsabilidades
- Flujo completamente streaming