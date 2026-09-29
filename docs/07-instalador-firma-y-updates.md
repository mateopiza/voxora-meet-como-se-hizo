# 7. Instalador, firma y auto-actualización

La meta: que compartir la app sea **un enlace a un solo `.exe` firmado**, y que después de la
primera instalación nadie tenga que volver a descargar nada a mano.

## 7.1 Un release = un comando

`npm run release` produce `VOXORA-Meet-Setup-<versión>.exe` a partir de una única fuente de
versión (el `package.json` raíz, del que salen el VERSIONINFO de todos los binarios, la versión
de la UI, el nombre del instalador y el manifiesto del canal de updates):

| Paso | Qué hace |
|---|---|
| build | Compila helpers WASAPI, cámara virtual, shell, desinstalador y el empaquetador |
| layout | Arma la carpeta instalable: binarios, motor en un solo archivo (esbuild) **ofuscado**, UI ofuscada módulo a módulo (imports intactos), y un Node LTS oficial verificado contra SHASUMS256 |
| firma interna | Firma cada `.exe`/`.dll` propio **antes** de empaquetar (el Node incluido conserva la firma de OpenJS) |
| package | El empaquetador propio comprime el layout (LZMS) con SHA256 y lo incrusta en el instalador |
| firma | Firma el instalador |
| verify | Verificación automática del resultado (abajo) |

## 7.2 Instalador propio en C++ (no NSIS, no WiX, no MSIX)

El instalador es un ejecutable C++ con WebView2 que renderiza un asistente con la identidad de
la marca: Bienvenida → Licencia → Ubicación → Progreso → Listo. ¿Por qué propio?

- Necesita **elevación real** y pasos que las plantillas genéricas hacen incómodos: registrar la
  DLL de la cámara virtual en HKLM (y que viva en una ruta legible por LocalService, es decir
  `Program Files`), crear la entrada de desinstalación, y explicar el estado del micrófono
  virtual al terminar.
- El mismo binario cubre todos los modos: asistente, **silencioso** (`/S`, usado por las
  actualizaciones, con relanzamiento de la app), desinstalación (conservando datos del usuario
  salvo purga explícita), **`/extract`** (extrae y verifica la carga útil sin tocar el sistema)
  y un modo `/preview` que recorre la UI simulada y captura pantallas — con el que se probó el
  asistente sin instalar nada cientos de veces.
- MSIX se descartó: el modelo de contenedor complica registrar la cámara virtual y el driver.

## 7.3 La firma como cadena de custodia

Todo lo propio se firma con Authenticode mediante un servicio de firma con la clave en un HSM en
la nube (no existe un `.pfx` en disco que robar). El detalle operativo con gracia: cada firma
pide un OTP y el servicio impone un intervalo mínimo entre firmas — un release completo son 8
firmas, así que `npm run release` se hace con el móvil en la mano. Incómodo y correcto: nadie
firma binarios de esta app sin el segundo factor.

Dos datos aprendidos por las malas sobre reputación en Windows:

- Un ejecutable **en ejecución no se puede firmar** (el archivo está bloqueado); el release
  exige cerrar la app o renombrar los exe bloqueados.
- **SmartScreen** no se fía de certificados OV nuevos: las primeras descargas muestran "Windows
  protegió su PC" hasta acumular reputación. Firmar siempre con el mismo certificado acelera el
  proceso; un EV lo elimina desde el primer día. Por eso jamás se comparte un build sin firmar.

## 7.4 `verify-release`: paranoia automatizada

El último paso del release verifica el artefacto como lo haría un auditor:

- Layout completo y limpio: sin `.pdb`, sin sourcemaps, sin tests, sin `node_modules`, sin `.env`.
- **Ningún JS legible**: compacto, sin comentarios, identificadores ofuscados.
- **Ninguna credencial** del entorno de build dentro del layout ni del instalador (búsqueda
  binaria real, no una lista de buenos deseos).
- VERSIONINFO correcto en todos los binarios; el Node incluido con su firma oficial; lo propio
  firmado con el thumbprint esperado.
- El motor empaquetado **arranca** con el Node incluido y responde `ping`; la UI ofuscada carga
  en Edge headless sin errores nuevos y con el grafo de módulos completo.
- `/extract` reproduce el layout **byte a byte**.

La regla de fondo: si el verificador no puede probarlo, el release no sale.

## 7.5 Auto-actualización sin backend

El canal de updates es un bucket S3 con tres objetos por versión: el instalador (inmutable), un
JSON de historial y un `latest.json` (sin caché) con versión, URL, SHA256, tamaño, notas y
versión mínima soportada.

En la app, el updater comprueba el canal a los 30 s de abrir y luego cada 6 h. Antes de aplicar
nada verifica **tamaño + SHA256 + Authenticode** — incluyendo que el firmante sea exactamente el
thumbprint esperado, no "cualquier firma válida" — y nunca actualiza durante una sesión de
doblaje. Al aceptar, relanza el instalador elevado en modo silencioso y la app se reabre sola.
`minSupportedVersion` permite marcar una actualización como obligatoria (la UI no ofrece "Más
tarde").

El script de publicación tiene sus propios frenos: se niega a subir un instalador sin firma, una
versión menor que la publicada o la misma versión con contenido distinto; y al terminar descarga
el canal **sin credenciales**, como lo haría una app cualquiera, y compara el SHA256. Un test de
integración recorre el updater real contra un servidor HTTP local con todos los casos hostiles:
firma ausente, firmante ajeno, SHA alterado, 404, update obligatoria, y la aplicación real con
relanzamiento.

## 7.6 Diagnóstico en producción

Logs rotados en el perfil local (`shell.log` con el stderr del motor incluido, `engine.log`,
`installer.log`) y minidumps si el shell cae. El botón "Abrir carpeta de registros" en la UI
existe por una razón: pedirle una carpeta a un usuario no técnico funciona; pedirle "el log"
no.
