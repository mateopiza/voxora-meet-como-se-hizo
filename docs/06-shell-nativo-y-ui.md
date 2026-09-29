# 6. Shell nativo y UI: Win32 + WebView2 en lugar de Electron

## 6.1 La decisión

La opción por defecto en 2026 para una app de escritorio con UI rica es Electron (o Tauri). Se
eligió otra cosa: **un ejecutable Win32 en C++17 con WebView2**, y la lógica de IA en un proceso
Node headless aparte.

Razones, en orden de peso:

1. **La app ya necesita C++ sí o sí**: captura de webcam con Media Foundation, ring de video,
   memoria compartida, WASAPI, detección de dispositivos. Con Electron habría hecho falta todo
   ese código nativo *además* de Chromium; el shell propio elimina la capa intermedia.
2. **Peso y arranque**: WebView2 usa el runtime Evergreen ya instalado en Windows (Edge está en
   todas partes); no se distribuyen ~200 MB de Chromium. El instalador final pesa lo que pesa el
   Node runtime + los binarios propios.
3. **El video no debe cruzar procesos de más**: el shell captura, retrasa y publica los frames
   sin serializarlos jamás (capítulo 3).
4. Electron seguía siendo la opción si la UI lo hubiera exigido — pero una UI de panel de
   control (pestañas, sliders, listas de dispositivos) no lo exige.

La UI en sí es **HTML/CSS/JS vanilla, sin frameworks ni CDN**: los tokens de diseño y las
fuentes (woff2) del design system de la marca se sirven en local. Cero `npm install` en la UI.

## 6.2 WebView2 bien cerrado

WebView2 por defecto es un navegador; convertirlo en una UI de app requiere cerrarlo:

- La carpeta de la UI se mapea con `SetVirtualHostNameToFolderMapping` a un host virtual
  (`https://app.voxora-meet/`) — **contexto seguro**, así `getUserMedia` funciona para la
  vista previa de la webcam y la prueba de mic sin levantar un servidor local.
- Navegación restringida al host propio; `window.open` y enlaces externos se abren en el
  navegador del sistema, y solo con esquemas `https:`/`http:`/`ms-settings:`.
- Permisos de cámara/micrófono concedidos únicamente al origen propio; todo lo demás, denegado.
- Autofill, guardado de contraseñas y barra de estado desactivados; DevTools solo en build
  Debug. CSP estricta de solo-recursos-propios (con la excepción puntual de `media-src https:`
  para las previews de voces de ElevenLabs).
- Las grabaciones locales (muestras de voz, pruebas de cámara) se exponen mediante hosts
  virtuales de solo lectura (`https://samples.voxora-meet/…`), nunca con rutas de archivo.
- Si el proceso del navegador cae, se recrea; si cae el renderer, se recarga. La app no muere
  por un fallo de WebView2.

El SDK de WebView2 va **vendorizado con enlace estático** del loader: no hay
`WebView2Loader.dll` que pueda ser secuestrada junto al exe.

## 6.3 El puente de tres actores

La UI habla con el shell por `postMessage`; el shell reenvía al motor por stdio. Un solo formato
de mensajes con sobres distintos:

| Dirección | Sobre |
|---|---|
| UI → shell → motor | `{ type: 'engine', id, cmd, params }` |
| motor → shell → UI | `{ type: 'engine-reply', id, ok, result \| error }` y `{ type: 'engine-event', event, data, engineNowMs }` |
| UI → shell (nativo) | `{ type: 'native', id, cmd, params }` (diálogos, logs, grabación de tomas…) |
| shell → UI | `{ type: 'devices', … }` cuando cambia el hardware |

Detalles que evitaron sufrimiento:

- **El shell observa los comandos que reenvía**: cuando ve pasar `session.start` con éxito,
  activa el delay de video; cuando ve `delay.set`, adopta el valor efectivo de la respuesta.
  La UI no coordina a los dos dueños del delay — el shell lo hace, en un solo lugar.
- **`engineNowMs`**: el shell sella cada evento del motor con su hora de llegada convertida al
  reloj del motor, y la UI calcula latencias por turno sin adivinar desfases entre relojes.
- **El motor puede morir**: el shell lo relanza con espera creciente (1→60 s) y publica
  `engine-state`; la UI muestra el estado y, tras 5 caídas seguidas, un botón de reinicio
  manual. Cualquier comando durante la caída responde `engine_unavailable` con el motivo.
- La detección de dispositivos (WM_DEVICECHANGE + IMMNotificationClient) vive en el shell:
  conectar unos auriculares o una webcam refresca las listas de la UI sin pedir "refrescar".

## 6.4 Los secretos, fuera del alcance de la UI

Las API keys de los proveedores se guardan cifradas con **DPAPI** (ligadas al usuario de
Windows) en el perfil local, separadas del `settings.json` plano. La UI nunca las ve completas;
el motor las usa y reporta solo su estado (`missing_key`, `provider_auth`, …).
