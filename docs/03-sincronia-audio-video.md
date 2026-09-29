# 3. Sincronía audio-video

El truco central del producto: **si el audio va a llegar tarde (porque doblarlo toma tiempo), que
el video llegue igual de tarde.** Un delay uniforme de 2–6 segundos es imperceptible en una
reunión — nadie nota que "el otro va 4 segundos detrás" — pero un desfase entre labios y voz de
medio segundo se nota inmediatamente y resulta molesto.

## 3.1 La línea de tiempo de audio (`SyncBuffer`, en el motor)

El `SyncBuffer` recibe tres flujos y produce uno:

- `pushAudio({ pcm, timestamp })` — el audio original del mic, continuo.
- `pushDub(dubResult)` — el doblaje de cada turno, cuando el pipeline lo termina
  (`sourceTimestamp` = inicio del turno original).
- un tick periódico que libera lo que ya cumplió `delayMs`.

La salida es **PCM 48 kHz continuo** hacia el micrófono virtual. Para cada rango de tiempo
liberado decide qué suena:

| Situación | Qué suena |
|---|---|
| El doblaje del turno llegó antes de que su rango cumpliera el delay | El doblaje (voz clonada) |
| El doblaje no llegó a tiempo | Original o silencio, según `fallbackMode` |
| Nadie habló | Silencio |

El delay es configurable **en caliente** (2000–6000 ms). Al subirlo no hay audio elegible durante
la diferencia (se emite silencio); al bajarlo se saltan los rangos más viejos. El remuestreo
16 kHz → 48 kHz (y de la salida del TTS a 48 kHz) es un resampler lineal propio, sin
dependencias.

## 3.2 El video no pasa por Node

El `SyncBuffer` tiene un `pushFrame()` funcional (con tests), pero el shell nativo **no lo usa**:
1280×720 RGBA a 30 fps son ~110 MB/s, inviables por un pipe JSON-lines. En su lugar, el shell
replica la misma política con un **ring de frames propio**, en C++:

1. `CameraCapture` lee la webcam con `IMFSourceReader` (RGB32, 1280×720@30) y sella cada frame
   con `QueryPerformanceCounter` **en el momento de captura**.
2. Los frames entran a un ring con capacidad para el delay máximo (6 s × fps + margen).
3. Un hilo de publicación, cada ~1/fps, publica en la memoria compartida de la cámara virtual el
   frame más nuevo con `timestamp <= now - delayMs` — el mismo criterio que el tick del
   `SyncBuffer`.
4. Los cambios de delay siguen la misma política que el audio: al subir, se mantiene el último
   frame publicado (congelado, igual que el audio emite silencio); al bajar, se saltan frames.

Fuera de la sesión de doblaje la cámara virtual no se apaga (comportamiento por defecto,
`cameraAlwaysOn`): el mismo ring publica con **retraso 0** — en vivo — y al iniciar sesión el
retraso pasa a `delayMs` sin cortar la cámara. Así Meet nunca ve la cámara "desaparecer" al
activar el doblaje.

## 3.3 Dos relojes, cero conversiones

El motor usa `performance.now()` (ms monotónicos); el shell usa QPC en unidades de 100 ns. La
tentación era establecer una correspondencia entre ambos relojes y convertir timestamps. No hace
falta, y evitarlo eliminó una clase entera de bugs:

> Cada lado retiene **con el mismo `delayMs` desde su propio instante de captura**. El mic y la
> webcam capturan al mismo hablante en el mismo instante real; si ambos lados retienen N ms desde
> su propia marca de captura, la salida queda sincronizada sin que ningún timestamp cruce el
> límite entre procesos.

La única regla necesaria es que **el shell sea el dueño del valor de `delayMs`**: cuando el
usuario mueve el control, el shell envía `delay.set` al motor **y** actualiza su ring en la misma
acción; el motor responde con el valor efectivo (clamp 2000–6000) y el shell lo adopta. Un solo
origen del dato, dos aplicaciones locales del mismo número.

## 3.4 Por qué frases + delay y no streaming palabra a palabra

Se evaluó doblar en streaming (traducir y sintetizar palabra por palabra, tipo intérprete
simultáneo). Se descartó por tres razones:

1. **Calidad**: traducir sin el final de la frase obliga a re-emitir correcciones o a traducir
   mal; los idiomas reordenan (el verbo alemán al final, la negación española antes del verbo…).
2. **Voz**: la prosodia del TTS por frase completa es radicalmente mejor que concatenar trozos.
3. **Sincronía**: con streaming, el desfase labios-audio es variable e incontrolable; con turno
   completo + delay uniforme, es constante y desaparece perceptualmente.

El precio — esos 2–6 segundos — es exactamente lo que el producto asume desde el nombre: es
doblaje profesional, no subtitulado en vivo.
