# 4. La cámara virtual

## 4.1 La API moderna que casi nadie usa todavía

Históricamente, una cámara virtual en Windows era un filtro DirectShow (frágil, por proceso) o un
driver de kernel AVStream (caro: firma, atestación, riesgo). Desde Windows 10 2004 existe una
tercera vía, sorprendentemente poco conocida: la **Media Foundation Virtual Camera API**
(`MFCreateVirtualCamera` + un media source que corre en el **Frame Server** del sistema).

- Corre en **modo usuario** — no hay driver de kernel que firmar ni atestar.
- La cámara aparece en *todas* las apps que usan el stack moderno de cámara (Chrome/Meet, Zoom,
  Teams, la app Cámara de Windows).
- La pieza propia es una **DLL COM** con el media source, que el sistema carga dentro del
  servicio Frame Server (que corre como **LocalService**), más un pequeño **host** (proceso que
  llama a `MFCreateVirtualCamera` y mantiene viva la cámara).

Comparado con el micrófono virtual (capítulo 5), esta pieza costó una fracción: mismo resultado
funcional, cero burocracia de firma de drivers. La asimetría es notable — Microsoft modernizó la
cámara virtual pero no ofrece un equivalente en modo usuario para crear un micrófono.

## 4.2 Transporte de frames: memoria compartida con seqlock

El shell produce frames (webcam + delay + efectos) y la DLL los consume desde otro proceso (el
Frame Server). El transporte es una sección de memoria compartida:

- Cabecera fija de 64 bytes + **3 slots** de frame RGBA8 1280×720.
- Cada slot usa un **seqlock**: el productor incrementa un contador de secuencia antes y después
  de escribir (impar = escribiendo); el consumidor lee y valida que la secuencia no cambió. Sin
  mutex entre procesos, sin bloquear al productor jamás — si el consumidor pierde un frame, lee
  el siguiente.
- Un **latido del productor** (timestamp QPC en unidades de 100 ns) permite a la DLL detectar
  que el shell murió y volver a la imagen de espera (el logo). Comparación directa con
  `MFGetSystemTime()`: es el mismo reloj subyacente (diferencia medida < 10 µs).

El mismo productor lo usan el shell y una herramienta CLI de pruebas, y un test E2E real registra
la DLL, arranca el host, escribe frames y los lee como lo haría una app de cámara.

## 4.3 El bug del "Meet solo ve la imagen de espera"

El mejor bug del módulo, porque solo aparecía en el caso de uso real:

- La sección de memoria la crea **la DLL** (en el Frame Server, con nombre `Global\…`), unos
  0,6 s **después** de que el host arranque la cámara. El shell no puede crearla él mismo:
  no tiene `SeCreateGlobalPrivilege` (LocalService sí lo tiene, las apps normales no), así que
  solo puede *abrirla* cuando ya existe.
- El shell la intentaba abrir una vez antes de lanzar el host (fallaba, obvio) y después solo
  reintentaba en la rama "no tengo frames que publicar". Con la webcam encendida siempre había
  frames, así que **nunca reintentaba**: la cámara virtual quedaba para siempre en la imagen de
  espera, pero solo si tu webcam funcionaba bien.
- Arreglo: reintentar la apertura cada 500 ms **siempre** que no se tenga el mapping, lleguen o
  no frames.

Moraleja: los reintentos condicionados a "estoy ocioso" son bombas de tiempo; la condición de
reintento debe ser "no tengo el recurso", y nada más.

## 4.4 Registro y ciclo de vida

- La DLL debe registrarse en HKLM y vivir en una ruta legible por LocalService — por eso el
  instalador pide elevación e instala en `Program Files` (capítulo 7).
- El host se lanza con un pipe a su stdin: si el shell muere, el host ve EOF y apaga la cámara.
  Nada de procesos huérfanos publicando una cámara congelada.
- Con el ajuste por defecto (`cameraAlwaysOn`) la cámara virtual existe mientras la app esté
  abierta — en vivo fuera de sesión, con delay durante el doblaje (capítulo 3) — para que en
  Meet siempre haya una "VOXORA Meet Camera" que seleccionar.

## 4.5 Efectos de imagen en el productor

Como el shell ya toca cada frame, los efectos de imagen salieron casi gratis y se aplican antes
de publicar: espejo, volteo, rotación (0/90/180/270), encuadre 16:9 ↔ 9:16 (recorte vertical
centrado con laterales difuminados, para verse "como un móvil" sin deformar), zoom 1–2× con
paneo, y brillo/contraste/saturación/temperatura. Todos son ajustes planos del settings store,
se aplican en caliente y viajan por el mismo puente UI → shell.
