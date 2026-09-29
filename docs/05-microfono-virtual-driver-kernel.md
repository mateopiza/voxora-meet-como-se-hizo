# 5. El micrófono virtual: un driver de kernel en 2026

La pieza más cara del proyecto, y la más instructiva. Para que Google Meet pueda seleccionar
"VOXORA Meet Microphone" como micrófono, ese dispositivo tiene que **existir** para Windows — y
crear un endpoint de audio nuevo sigue exigiendo, en 2026, un driver de kernel.

## 5.1 Por qué no había atajo

- Un **APO** (Audio Processing Object, modo usuario) solo procesa el audio de un endpoint **que
  ya existe**; no puede crear un micrófono nuevo. Descartado.
- Las vías reales para crear un endpoint: driver PortCls/WaveRT, driver AVStream KS, o emular
  hardware Bluetooth/USB. **WaveRT** es la vía estándar — es lo que usan VB-Cable y Voicemeeter —
  y es compatible con el modo event-driven de WASAPI.
- "WaveRT" suena a hardware con DMA, pero en un cable virtual no hay DMA: el "hardware" es un
  buffer cíclico compartido con el motor de audio de Windows y una **posición virtual que avanza
  con el reloj del sistema**.

El driver (`voxorameet.sys`, ~28 KB) expone dos endpoints conectados por un anillo interno:

```
app ──PCM──▶ helper WASAPI ──▶ [VOXORA Meet Speaker]  (render)
                                       │   anillo float32 48 kHz en kernel
                                       ▼   (latencia acotada a 100 ms)
Meet/Zoom ◀──── WASAPI ◀────── [VOXORA Meet Microphone]  (captura)
```

Cuatro subdispositivos (wave + topología por cada endpoint), nodos de volumen/mute para que el
panel de sonido muestre el slider, jack "siempre conectado", y conversión
s16↔float / mono↔estéreo / remuestreo 16–48 kHz en el anillo. Un timer de kernel de 10 ms mueve
el audio y dispara los eventos de notificación del modo pull.

## 5.2 Compilar un driver sin instalar el WDK

El hallazgo más reutilizable del proyecto. La vía oficial (instalar Visual Studio + WDK + la
extensión de VS) es pesada y contamina la máquina. Desde hace unos años Microsoft publica el
**WDK y el SDK como paquetes NuGet**, y con ellos se puede compilar un driver con un script:

1. Descargar `Microsoft.Windows.WDK.x64`, `Microsoft.Windows.SDK.CPP` y
   `Microsoft.Windows.SDK.CPP.x64` (misma versión, p. ej. 10.0.26100.x) de nuget.org y
   extraerlos con `tar.exe` — son ~320 MB, sin instalador, sin administrador.
2. Cargar `vcvars64.bat` del MSVC ya instalado (vía `vswhere`) y **sustituir** `INCLUDE`/`LIB`
   por las rutas de kernel del WDK (`km\crt`, `km`, `shared`, `km\x64`) — lo mismo que hace el
   toolset `WindowsKernelModeDriver10.0` de MSBuild.
3. Invocar `cl`/`rc`/`link` con los flags copiados de los `.props` del WDK: `/kernel /guard:cf
   /Zp8 /GR- …` y `link /DRIVER /KERNEL /SUBSYSTEM:NATIVE /ENTRY:GsDriverEntry /NODEFAULTLIB`
   contra `portcls.lib`, `stdunk.lib`, `ntoskrnl.lib`, etc.
4. Rematar con las herramientas del propio paquete: `stampinf` (versión del INF), `infverif /h`
   (las reglas que aplica Partner Center), `Inf2Cat`, `ApiValidator` y `makecab` para el paquete
   de atestación.

Resultado: cualquier máquina con Visual Studio C++ puede compilar el driver con un comando,
reproducible, sin tocar el sistema. El mismo script corre el Code Analysis del WDK con el plugin
de drivers.

## 5.3 Lo que el compilador no te dice (y el kernel sí)

Compilar `/W4 /WX` limpio es el principio, no el final. Las correcciones más valiosas fueron de
revisión, no de compilador:

- **Código paginable con spinlock tomado**: varias funciones en `#pragma code_seg("PAGE")`
  tomaban un spinlock — ejecutar código paginable a `DISPATCH_LEVEL` es un BSOD
  (`IRQL_NOT_LESS_OR_EQUAL`) esperando su momento. Pasaron a sección no paginada con
  `ASSERT(PASSIVE_LEVEL)` donde correspondía, y un slot de puntero pasó de spinlock a
  `InterlockedCompareExchangePointer`.
- **Volumen multicanal**: anunciar el soporte de volumen con un solo rango sin el flag
  multicanal hace que el motor de audio ajuste solo el canal 0 — la ganancia media queda a la
  mitad en estéreo. Hay que devolver un rango por canal con el flag
  `…_BASICSUPPORT_MULTICHANNEL`.
- **`ExAllocatePoolWithTag` está obsoleta**: el helper clásico `stdunk.h` define `operator new`
  sobre ella; se redefinieron los operadores de kernel sobre **`ExAllocatePool2`** (que además
  cero-inicializa y exige Windows 10 2004+ — y por coherencia, el INF declara ese mínimo).
- Un `Write` con `SampleRate == 0` entraba en bucle infinito **con el spinlock tomado** (cuelgue
  total del motor de audio). Guardas de parámetros también en kernel, siempre.
- SAL por todas partes: anotaciones de IRQL coherentes, `_Dispatch_type_`, tamaños de buffers.
  El Code Analysis del WDK con las reglas de drivers quedó en 0 avisos.

Validación estática final: `infverif /h` VALID (las reglas de firma de Microsoft), Inf2Cat sin
errores, ApiValidator "All binaries are Universal", y `dumpbin` confirmando NX, CFG, sin
secciones RWX e importando solo `portcls.sys`/`ntoskrnl.exe`/`HAL.dll`.

## 5.4 Desktop vs DCH: una decisión consciente

El INF es de tipo *Desktop*, no DCH: registra nombres de pin propios en
`HKLM\…\MediaCategories`, que es lo que hace que el dispositivo se llame exactamente
"VOXORA Meet Microphone" en todas partes. `infverif /w` (reglas DCH) lo marca como error;
`infverif /h` (lo que exige la firma de atestación) lo acepta. Mientras la distribución sea el
instalador propio y no Windows Update, el nombre bonito gana.

## 5.5 La burocracia: atestación, EV y la trampa del sole proprietor

Un driver de kernel x64 **debe** estar firmado por Microsoft para cargar en un Windows normal.
La vía sin certificación HLK completa es la **firma por atestación**: subir un `.cab` firmado a
Partner Center y recibir el paquete firmado por Microsoft.

Los aprendizajes de esa burocracia, que ninguna documentación cuenta junta:

- **Partner Center exige un certificado EV solo para registrar la cuenta** de hardware. Después
  del registro, los `.cab` pueden firmarse con un certificado OV registrado. Es decir: el EV es
  el peaje de entrada, no el costo recurrente.
- Una **persona natural** (sin empresa) sí puede: la vía es el EV "sole proprietor" que ofrecen
  algunas CA. Poco documentado y fácil de descartar por error como "solo para empresas".
- Para desarrollo, nada de esto bloquea: en una **VM con testsigning** (Secure Boot desactivado,
  certificado autofirmado, `bcdedit /set testsigning on`) el driver carga, los endpoints
  aparecen, y Driver Verifier lo somete a estrés. La regla autoimpuesta: kernel no probado solo
  se carga en VMs con checkpoint, jamás en la máquina de trabajo.

## 5.6 El puente hacia Node

Node no habla WASAPI, así que el motor entrega el doblaje mediante un helper nativo
(`wasapi-render.exe`, C++ sin dependencias): lee PCM s16le por stdin y lo rinde en el endpoint
"VOXORA Meet Speaker" con `AUDCLNT_STREAMFLAGS_AUTOCONVERTPCM`, modo compartido event-driven y
contrapresión sobre stdin (cola máxima de 2 s). La clase `VirtualMic` de Node envuelve el
proceso: `open()` resuelve cuando WASAPI está rindiendo, `write(pcm)` respeta la contrapresión,
`close()` vacía la cola.

Mientras el driver propio recorre la burocracia de firma, la misma clase detecta VB-Cable
("CABLE Input") como respaldo — el producto funciona hoy, y el driver propio elimina la
dependencia externa cuando Microsoft devuelva el paquete firmado.
