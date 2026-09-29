# 8. Lecciones aprendidas

Lo que este proyecto confirmó, lo que enseñó y lo que se haría distinto.

## Lo que volvería a hacer igual

**Definir los contratos antes que los módulos.** Un documento fuente-de-verdad con los eventos,
formas y unidades de cada frontera (capture→pipeline→sync→entrega, shell⇄motor) permitió
construir módulos en paralelo, testearlos aislados y cambiar implementaciones sin tocar vecinos.
Cada ambigüedad que el contrato resolvió por escrito fue un bug que no existió.

**Cero dependencias de runtime en el motor.** Escribir a mano WAV, resampling y HTTP con
reintentos costó unas horas; a cambio: sin `node_modules` que auditar ni empaquetar, sin supply
chain en el binario distribuido, y tests (`node --test`) que corren en cualquier parte. En una
app que se firma y distribuye, cada dependencia es superficie de ataque y de rotura.

**Inyección de dependencias en todo lo que toca el mundo.** Mic, red, procesos y relojes entran
inyectados. Resultado: ~200 tests que corren sin hardware, sin red y sin driver, y un puñado de
pruebas separadas y explícitas contra proveedores reales (que gastan dinero y se ejecutan a
propósito, no en cada `npm test`).

**El delay como característica, no como vergüenza.** Aceptar 2–6 s de retraso y sincronizar el
video con él convirtió la mayor debilidad del doblaje (la latencia) en la base de la experiencia
(labios en sincronía). Pelear contra la física — streaming palabra a palabra — habría dado un
producto peor en calidad, voz y sincronía a la vez.

**El shell observa, la UI no coordina.** Todo lo que necesita pasar en dos sitios a la vez (el
delay en audio y video, el ciclo de vida de la cámara) lo coordina un solo actor viendo pasar
los comandos. Cuando dos componentes deben moverse juntos, la respuesta no es un tercer mensaje:
es un solo dueño.

**Errores con código estable y mensaje listo para humanos.** Distinguir `provider_payment` de
`provider_auth` de `model_unavailable` (nombrando modelo y etapa) convierte fallos opacos de
proveedor en mensajes accionables. El catálogo de modelos en vivo nació de un modelo retirado
por el proveedor que rompió otro producto en silencio.

## Lo que enseñó este proyecto

**En Windows, video y audio virtuales viven en décadas distintas.** La cámara virtual es una API
moderna de modo usuario (Media Foundation, sin firma); el micrófono virtual exige un driver de
kernel firmado por Microsoft con burocracia de Partner Center. Presupuestar ambos "igual porque
son parecidos" habría sido un error de meses.

**El WDK de NuGet existe y cambia el juego.** Compilar un driver de kernel reproduciblemente,
sin instalar el WDK, sin administrador, con un script — incluyendo `infverif`, Inf2Cat, Code
Analysis y el `.cab` de atestación — es posible y casi nadie lo documenta junto.

**El código de kernel se revisa con otras gafas.** Los bugs graves del driver no los vio el
compilador: código paginable bajo spinlock, un bucle infinito con spinlock tomado ante un
parámetro cero, volumen multicanal mal anunciado. En kernel, el equivalente de "se me pasó" es
un pantallazo azul en la máquina de un cliente; VM con checkpoint, testsigning y Driver Verifier
no son opcionales.

**Los reintentos condicionados a "estoy ocioso" son bombas de tiempo.** El bug de la cámara que
solo fallaba con la webcam funcionando (capítulo 4) dejó la regla: se reintenta mientras falte
el recurso, sin más condiciones.

**La reputación en Windows es un activo que se cultiva.** SmartScreen, la firma con HSM y OTP,
el updater que verifica el thumbprint exacto: distribuir software de escritorio en 2026 es
gestionar una cadena de confianza, y cada eslabón (¿quién firma?, ¿qué verifica el que
descarga?, ¿qué puede inyectarse dónde?) hay que decidirlo a propósito.

**Verificar el artefacto, no el proceso.** El paso `verify-release` (¿hay JS legible?, ¿hay
credenciales dentro?, ¿arranca el motor empaquetado?, ¿`/extract` es byte a byte?) atrapó
errores que ningún "el build pasó" habría visto.

## Lo que se haría distinto

- **Empezar la burocracia de firma el día 1.** El código del driver estuvo listo en días; el
  registro en Partner Center (que exige certificado EV solo para crear la cuenta) es un trámite
  externo de semanas que no se paraleliza solo. Todo lo que dependa de terceros con SLA humano
  debe arrancar antes que el código que lo necesita.
- **Probar el driver en VM más temprano.** El driver compiló y validó estáticamente limpio,
  pero un driver que nunca cargó es una hipótesis, no un componente. El hueco entre "compila
  perfecto" y "sobrevive a Driver Verifier" debió cerrarse la misma semana.
- **Presupuestar la fricción de firmar.** Ocho OTP con 36 s de espera entre firmas por release
  suena a anécdota hasta que se hacen tres releases en una tarde. La firma con segundo factor es
  la decisión correcta; automatizar todo alrededor de ella (colas, reintentos, orden de firmado)
  merece inversión desde el principio.

## El resumen del resumen

El pipeline de IA — la parte "mágica" — fue la parte fácil: proveedores maduros, APIs claras,
todo testeable. Lo difícil, y donde vivió casi toda la ingeniería, fue **el último centímetro**:
aparecer como una cámara y un micrófono legítimos ante Windows, mover audio y video en sincronía
entre cuatro procesos y dos relojes, y entregar todo eso en un `.exe` firmado que se actualiza
solo. La IA es el motor; el producto es la carrocería, y la carrocería era Windows.
