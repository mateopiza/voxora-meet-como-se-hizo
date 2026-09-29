# VOXORA Meet — cómo se hizo

**Doblaje profesional en tiempo real con tu propia voz clonada, para Google Meet, Zoom o Teams.**
Hablas en tu idioma y la reunión te escucha en el idioma destino — con tu voz, y con el video
sincronizado con el audio doblado.

Este repositorio documenta **cómo se construyó** VOXORA Meet: la arquitectura, las decisiones
técnicas (y las descartadas), y los problemas reales que aparecieron al construir una app de
Windows que combina IA de voz con drivers de kernel. No es el código fuente del producto; es el
diario de ingeniería.

## La idea en 30 segundos

En una videollamada no puedes "subtitular" tu voz: la otra persona necesita **escucharte**. Los
traductores en vivo existentes usan voces genéricas y desincronizan los labios del audio. VOXORA
Meet toma otro camino:

1. Captura tu micrófono y corta el audio por **frases completas** (detección de silencio), no por
   ventanas fijas de tiempo — más contexto por turno = mejor transcripción y traducción.
2. Transcribe con **Whisper large-v3** (Groq), traduce con un LLM con **memoria de la
   conversación y glosario propio**, y sintetiza con **tu voz clonada** (ElevenLabs).
3. Retiene el **video de tu webcam el mismo tiempo** que tarda el doblaje (2–6 s configurables) y
   libera audio y video juntos: la reunión te ve hablando *en sincronía* con el audio traducido.
4. Se presenta ante Meet/Zoom/Teams como una **cámara virtual** y un **micrófono virtual** de
   Windows — no requiere OBS ni extensiones del navegador.

```
mic ──▶ VAD por frase ──▶ STT (Groq Whisper) ──▶ Traducción (Groq LLM     ──▶ TTS (ElevenLabs,
        (endpointing         verbose_json,          + memoria de turnos        voz clonada)
         por silencio)       anti-alucinación)      + glosario + tono)              │
                                                                                    ▼
webcam ──▶ ring de frames (delay = el del audio) ──▶ cámara virtual      buffer de sincronía
                                                     (Media Foundation)  (delay 2–6 s, fallbacks)
                                                                                    │
Google Meet ◀── "VOXORA Meet Microphone" ◀── driver de kernel WaveRT ◀── PCM 48 kHz ┘
```

## Lo que hace interesante este proyecto

- **Un driver de audio de kernel para Windows** (PortCls + WaveRT, el mismo enfoque de VB-Cable),
  compilado **sin instalar el WDK** — con los paquetes NuGet oficiales de Microsoft, sin
  administrador. [Capítulo 5](docs/05-microfono-virtual-driver-kernel.md).
- **Una cámara virtual en modo usuario** con la Media Foundation Virtual Camera API (sin driver
  firmado), memoria compartida con seqlock y un bug de sincronización que solo aparecía con la
  webcam encendida. [Capítulo 4](docs/04-camara-virtual.md).
- **Shell nativo Win32 + WebView2 en lugar de Electron**: la UI es HTML/CSS/JS vanilla servida en
  local, y el pipeline de IA corre en un motor Node headless controlado por JSON-lines sobre
  stdio. [Capítulo 6](docs/06-shell-nativo-y-ui.md).
- **Sincronía labios-audio con dos relojes distintos** (QPC en C++ y `performance.now()` en Node)
  sin convertir timestamps entre ellos. [Capítulo 3](docs/03-sincronia-audio-video.md).
- **Instalador propio en C++** (nada de NSIS/WiX), firma Authenticode con HSM en la nube, y
  auto-actualización que verifica firma + SHA256 antes de aplicar.
  [Capítulo 7](docs/07-instalador-firma-y-updates.md).

## Índice

| Capítulo | Contenido |
|---|---|
| [1. Arquitectura general](docs/01-arquitectura.md) | Módulos, contratos entre capas, por qué es un proyecto separado |
| [2. El pipeline de doblaje](docs/02-pipeline-de-doblaje.md) | VAD por frase, STT, traducción con contexto, TTS con voz clonada |
| [3. Sincronía audio-video](docs/03-sincronia-audio-video.md) | El buffer de delay, fallbacks, y el problema de los dos relojes |
| [4. La cámara virtual](docs/04-camara-virtual.md) | Media Foundation Virtual Camera, memoria compartida, efectos de imagen |
| [5. El micrófono virtual](docs/05-microfono-virtual-driver-kernel.md) | Driver WaveRT de kernel, compilar sin WDK, firma por atestación |
| [6. Shell nativo y UI](docs/06-shell-nativo-y-ui.md) | Win32 + WebView2 vs Electron, el puente UI ⇄ motor |
| [7. Instalador, firma y updates](docs/07-instalador-firma-y-updates.md) | Release de un solo archivo, Authenticode, canal de auto-update |
| [8. Lecciones aprendidas](docs/08-lecciones-aprendidas.md) | Lo que volvería a hacer igual y lo que no |

## Stack

| Capa | Tecnología |
|---|---|
| Motor de doblaje | Node.js ≥ 22, ESM puro, **cero dependencias de runtime**, tests con `node --test` |
| STT | Groq — Whisper `whisper-large-v3`, turno completo, `verbose_json` |
| Traducción | Groq — chat completions (`openai/gpt-oss-120b` por defecto, seleccionable) |
| TTS + clonación | ElevenLabs — `eleven_multilingual_v2` con Instant/Professional Voice Cloning |
| Shell de la app | C++17 Win32 + WebView2 (CMake + MSVC), UI HTML/CSS/JS vanilla sin frameworks |
| Cámara virtual | Media Foundation Virtual Camera API (modo usuario) |
| Micrófono virtual | Driver de kernel PortCls/WaveRT (`voxorameet.sys`), WDK 10.0.26100 vía NuGet |
| Captura de audio | WASAPI (helpers nativos propios en C++) |
| Instalador | C++ + WebView2, empaquetado LZMS propio, firma Authenticode |

## Estado

El proyecto se desarrolló de forma intensiva en septiembre de 2026. Los ocho milestones de código
están completos y verificados con proveedores reales; queda pendiente lo que no depende del
código: la atestación del driver de kernel en Microsoft Partner Center (requiere registro con
certificado EV) y la prueba end-to-end en una reunión real con el driver firmado. Mientras tanto
la app funciona con VB-Cable como micrófono virtual.

---

© 2026 VØXORA. Esta documentación se publica para compartir el proceso de construcción; el código
del producto no es open source. Los fragmentos de código incluidos son ilustrativos.
