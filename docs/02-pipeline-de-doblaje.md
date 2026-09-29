# 2. El pipeline de doblaje

La unidad de trabajo de todo el sistema es el **turno**: una frase completa del hablante, cerrada
por silencio. Esta decisión — aparentemente pequeña — es la que más calidad aportó.

## 2.1 Captura y VAD por frase

Los pipelines de baja latencia trocean el audio en ventanas fijas (p. ej. 2 s) y transcriben cada
trozo. Eso corta frases por la mitad: el STT pierde contexto, la traducción recibe fragmentos
sin sentido completo, y el TTS entona como un robot leyendo telegramas.

VOXORA Meet hace **endpointing por silencio**: un detector de actividad de voz (VAD) sobre la
energía de la señal decide cuándo empezó y cuándo terminó una frase, y emite el turno entero:

```
evento 'turn' → { pcm: Buffer (s16le mono 16 kHz), sampleRate: 16000,
                  startedAt, endedAt, voicedMs, rmsDb }
```

- La captura real la hace un **helper WASAPI nativo** (C++, sin dependencias) que escribe PCM
  crudo por stdout; Node lo consume como un stream. El segmentador de frases es JS puro y
  testeable con señales sintéticas.
- El costo es latencia (hay que esperar el fin de la frase), pero este producto la convierte en
  virtud: el delay es explícito y el video espera lo mismo (capítulo 3).
- La UI recibe además un evento `level` (~20 Hz) con `{ rmsDb, speaking }` para el vúmetro.

## 2.2 STT: Groq Whisper large-v3, turno completo

Cada turno se envía como WAV a Groq (`whisper-large-v3`) vía REST, con dos detalles que importan:

- **`verbose_json`**: la respuesta trae `no_speech_prob` por segmento. Whisper alucina texto
  cuando recibe respiraciones, teclados o silencio ("Subtítulos realizados por…"); filtrar por
  probabilidad de no-voz descarta esos turnos antes de que lleguen a la traducción. Un turno
  descartado resuelve `null` y no cuesta traducción ni TTS.
- **El campo `prompt` como vocabulario custom**: nombres propios, siglas y jerga del usuario se
  inyectan en el prompt de Whisper, que los usa como sesgo léxico. El usuario mantiene su lista
  de términos desde la UI y aplica desde el siguiente turno, sin reiniciar la sesión.

Se eligió Groq por velocidad de inferencia (el turno completo se transcribe en una fracción del
delay disponible) y porque un solo proveedor cubre STT y traducción con la misma API key.

## 2.3 Traducción: LLM con memoria, glosario y tono

La traducción usa chat completions de Groq (`openai/gpt-oss-120b` por defecto, seleccionable
desde la UI entre los modelos vivos de la cuenta). No es "traducir la frase": es traducir **esta
frase, en esta conversación, para este hablante**:

- **Memoria de turnos**: los últimos N pares transcripción→traducción (configurable, 0–64,
  default 8) viajan como contexto. Consigue consistencia terminológica: si "board" se tradujo
  como "junta directiva" en el turno 3, no se convierte en "tablero" en el 12.
- **Glosario persistente por usuario**: pares término→traducción (o "no traducir") que se
  inyectan en el prompt. Hasta 500 entradas; se releen en cada turno, así que editar el glosario
  a mitad de reunión aplica al turno siguiente.
- **Control de tono**: registro formal/profesional en las instrucciones del sistema.
- **`reasoning_effort`**: los modelos que lo soportan (familia gpt-oss, Qwen3) aceptan un nivel
  de razonamiento; `low` por defecto — suficiente para traducir y más barato/rápido. La escala se
  traduce entre familias (Qwen3 solo tiene `none`/`default`) y se omite en modelos que no la
  admiten.

Detalle de robustez: si el proveedor rechaza un parámetro opcional (`reasoning_effort`,
`language_code`…), el cliente reintenta una vez sin él **y recuerda no volver a enviarlo para ese
modelo** durante la sesión.

## 2.4 TTS: la voz del propio usuario

La síntesis usa ElevenLabs `eleven_multilingual_v2` con una **voz clonada** del usuario
(Instant Voice Cloning con ~1 minuto de muestras; Professional para quien quiera más fidelidad).

- El onboarding de clonación vive en la app: graba tomas guiadas con la webcam/mic reales
  (o importa .wav/.mp3/.m4a/.ogg/.flac), valida que sumen al menos 60 s y crea la voz vía API.
  La voz recién clonada queda seleccionada.
- Se descartó `eleven_v3` en el arranque: más costo y latencia sin necesidad comprobada, y sus
  voice settings funcionan distinto (presets de estabilidad en vez de valores continuos). El
  código igual lo soporta — el catálogo de modelos es dinámico — y normaliza los settings según
  las capacidades de cada modelo (`style`, `speaker_boost`, `speed`, normalización de texto).
- El TTS devuelve PCM (24/44.1/48 kHz según cuenta); el buffer de sincronía lo remuestrea a
  48 kHz para el micrófono virtual.

## 2.5 El catálogo de modelos es en vivo

En vez de hardcodear modelos, el motor consulta `GET /models` de Groq y ElevenLabs (caché de
10 min) y arma un catálogo con precio, ventana de contexto, niveles de razonamiento y
capacidades TTS por modelo. Sin red, cae a un catálogo estático marcado `offline: true`.

Esto salió de una herida real: un modelo de Groq usado en otro producto de la casa
**fue retirado del proveedor** y el producto empezó a fallar en silencio. VOXORA Meet marca
`available: false` si el modelo guardado en los ajustes ya no existe en el catálogo vivo, la UI
lo muestra, y el error `model_unavailable` nombra la etapa y el id en un mensaje listo para
mostrar al usuario.

## 2.6 Errores como contrato

Todos los errores del pipeline se normalizan a códigos estables con mensaje en español listo
para la UI: `missing_key`, `provider_auth`, `provider_payment`, `provider_quota`,
`model_unavailable`, `network`, `virtual_mic_missing`, `voice_missing`, `samples_too_short`,
`bad_request`, `internal`.

El caso `provider_payment` también viene de la vida real: una cuenta de ElevenLabs con un pago
rechazado devuelve 401 con `payment_issue`, y "tu API key es inválida" habría sido un
diagnóstico falso. Distinguir *auth* de *payment* de *quota* ahorra tickets de soporte.

## 2.7 Costos por turno

Cada `DubResult` trae `cost: { stt, translate, tts, totalVox }` calculado con los modelos que
**realmente** atendieron el turno (pueden cambiar a mitad de sesión). Un `SessionMeter` acumula,
avisa al cruzar un umbral y corta en el límite por sesión. La UI ofrece además un estimador:
minutos de reunión × proporción hablada × modelos elegidos → créditos/hora y USD/hora por etapa,
para que el usuario vea el efecto de cambiar de modelo antes de la reunión.
