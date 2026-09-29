# 1. Arquitectura general

## Por qué un proyecto nuevo y no un modo más de un producto existente

VOXORA ya tenía un pipeline de doblaje en producción para streaming en vivo (un plugin para OBS
con su propia API de doblaje). La tentación obvia era añadirle un "modo reuniones". Se descartó,
y esa decisión definió todo lo demás.

El pipeline de streaming está afinado para **latencia mínima**: ventanas de audio cortas, un solo
paso de traducción, voces genéricas rápidas (ElevenLabs Flash/Turbo). Una reunión profesional
necesita exactamente lo contrario:

| Dimensión | Streaming en vivo | Reunión (VOXORA Meet) |
|---|---|---|
| Prioridad | Latencia mínima | Calidad y fidelidad de voz |
| Unidad de trabajo | Chunk de tiempo fijo | Frase completa (VAD por silencio) |
| Traducción | Un paso, sin memoria | Memoria de N turnos + glosario + control de tono |
| Voz | Genérica rápida | Clonada del usuario (`eleven_multilingual_v2`) |
| Desfase labios-audio | Invisible en un stream | Inaceptable en una reunión → delay controlado |
| Entrega | OBS (v4l2loopback / PulseAudio) | Cámara y micrófono virtuales propios de Windows |
| Costos | Por stream | Créditos por minuto de reunión |

Un "perfil de configuración" no alcanzaba: cambia la unidad de trabajo, la capa de entrega
completa y el modelo de costos. Compartir código habría acoplado dos productos con requisitos
opuestos. Se reutilizaron **conceptos** ya probados (captura, pipeline STT→traducción→TTS,
dispositivos virtuales), no código.

## Los dos procesos

La app son dos procesos con responsabilidades disjuntas:

```
VoxoraMeet.exe (shell nativo, C++17 Win32)          engine (Node ≥ 22, headless)
┌─────────────────────────────────────┐   stdio    ┌──────────────────────────────────┐
│ ventana + bandeja + UI (WebView2)   │◀──────────▶│ MicCapture (VAD por frase)       │
│ captura de webcam (MF SourceReader) │ JSON-lines │ DubbingPipeline (STT→trad→TTS)   │
│ ring de video con delay             │            │ SyncBuffer (línea de tiempo)     │
│ memoria compartida → cámara virtual │            │ VirtualMic (→ driver / VB-Cable) │
│ detección de dispositivos           │            │ billing (créditos por sesión)    │
│ auto-updater                        │            │ settings + glosario + vocabulario│
└─────────────────────────────────────┘            └──────────────────────────────────┘
```

- **El shell es dueño del video**: captura la webcam, aplica el delay con un ring de frames y
  publica en la cámara virtual. Pasar video crudo (1280×720 RGBA a 30 fps ≈ 110 MB/s) por un pipe
  JSON habría sido inviable; el video nunca sale del proceso nativo.
- **El motor es dueño del audio**: VAD → STT → traducción → TTS → buffer de sincronía → micrófono
  virtual. Todo en Node, testeable sin hardware.
- Ambos aplican **el mismo `delayMs`**, cada uno con su propio reloj (capítulo 3).

La comunicación es **JSON-lines sobre stdio**: petición `{ id, cmd, params }`, respuesta
`{ id, ok, result | error: { code, message } }`, más eventos asíncronos
(`level`, `transcript`, `translation`, `dub`, `cost`, `status`…). Sin sockets, sin IPC exótico:
si el shell muere, el motor ve EOF y sale; si el motor muere, el shell lo relanza con espera
creciente (1, 3, 10, 30, 60 s).

## Módulos

| Módulo | Qué hace |
|---|---|
| `capture/` | Helper WASAPI nativo (mic → PCM s16le 16 kHz por stdout) + VAD por frase en JS |
| `pipeline/` | Clientes Groq (Whisper, chat) y ElevenLabs; orquestador del turno |
| `sync-buffer/` | Línea de tiempo de audio con delay 2–6 s; mezcla doblaje/original/silencio |
| `windows-camera/` | DLL de cámara virtual MF + host + productor de frames por memoria compartida |
| `windows-driver/` | Driver de kernel WaveRT (mic virtual) + helper de render WASAPI + firma |
| `billing/` | Tabla de créditos por modelo y medidor de sesión con límites |
| `app/engine/` | Motor headless: protocolo, controlador de sesión, settings |
| `app/native-shell/` | VoxoraMeet.exe: ventana, WebView2, video, dispositivos, updater |
| `app/ui/` | HTML/CSS/JS vanilla servido en local por WebView2 |
| `installer/` | Instalador C++ + WebView2 con empaquetado propio |

## Contratos entre capas

Los contratos viven en un documento fuente-de-verdad y los módulos solo se conocen por ellos:

- **capture → pipeline**: evento `turn` con `{ pcm (s16le mono 16 kHz), startedAt, endedAt,
  voicedMs, rmsDb }`. Un turno = una frase cerrada por silencio, nunca una ventana fija.
- **pipeline → sync-buffer**: `processTurn(turn)` resuelve `{ audioDub, sampleRate,
  sourceTimestamp, readyAt, transcript, translation, cost }` — o `null` si el turno se descartó
  (sin voz o alucinación de Whisper).
- **sync-buffer → entrega**: el buffer retiene `delayMs` y libera audio continuo hacia el
  micrófono virtual; el doblaje entra si llegó a tiempo para su rango, si no, original o
  silencio según el modo de fallback.
- **shell ⇄ engine**: los comandos JSON-lines del protocolo (sesión, dispositivos, voces,
  modelos, glosario, costos). El protocolo creció en versiones compatibles hacia atrás (v2:
  dispositivos y voces; v3: catálogo de modelos en vivo y estimación de costos).

Dos reglas transversales que pagaron dividendos:

1. **Todo tiempo es un timestamp monotónico en ms** (`performance.now()` en Node, QPC en C++).
   Nada de `Date.now()`: un ajuste de reloj del sistema a mitad de reunión no puede romper la
   sincronía.
2. **Toda fuente es inyectable**: `MicCapture` acepta cualquier `Readable` de PCM, el pipeline
   acepta `fetch` inyectado, `VirtualMic` acepta `spawn` inyectado. Los ~200 tests de los módulos
   JS corren con `node --test` sin red, sin hardware y sin driver.

## Convenciones

- Node ≥ 22, ESM puro, **cero dependencias de runtime** (fetch y WebSocket son globales; WAV,
  resampling y HTTP con reintentos están escritos a mano). Las únicas dependencias son de build
  (esbuild y un ofuscador, capítulo 7).
- C++17 con CMake + MSVC para todo lo nativo; el driver de kernel tiene su propio build
  (capítulo 5).
- Audio interno: PCM s16le mono; 16 kHz hacia el STT, 48 kHz hacia el micrófono virtual.
