+++
title = "Por qué los backup pins y la rotación importan en producción"
translationKey = "android-pinning-backup-pins-rotation"
date = "2026-11-13"
author = "Thiago Siqueira"
tags = ["android", "ssl-pinning", "mobile", "pki", "operaciones"]
keywords = ["android", "certificate pinning", "backup pin", "rotación de claves", "network security config"]
description = "Reproduzco la rotación de la clave del servidor para mostrar por qué un cliente que solo conoce el pin antiguo pierde la conexión, mientras otro preparado con un backup pin sigue funcionando, y qué exige esa continuidad a la operación."
showFullContent = false
readingTime = true
hideComments = false
+++

Una app logra conversar con su servidor, recibe la respuesta esperada y aplica correctamente
certificate pinning. Algún tiempo después, el equipo reemplaza el certificado TLS del servidor.
El dominio sigue siendo el mismo, el nuevo certificado es válido y el cambio es legítimo. Aun
así, la app deja de conectar.

¿Qué está rechazando?

En los artículos anteriores, investigo la confianza en TLS (Transport Layer Security), separo
las fallas de trust store de las fallas de pinning y muestro cómo una verificación local puede
ser identificada y alterada. Ahora quiero mirar el control durante un cambio normal del
servicio, mientras la app sigue ejecutando su política original.

La pregunta que cierra la serie es: *¿cómo rotar la clave legítima del servidor sin perder la
conexión con los clientes ya instalados?*

Para responder, sigo una rotación de clave y comparo el comportamiento de dos clientes. El
experimento aporta la evidencia, pero la pregunta es más amplia: una decisión de confianza
grabada en la app necesita seguir siendo compatible con un servidor que va a cambiar con el
tiempo.

## La conexión funciona; ¿qué espera encontrar exactamente la app?

Empiezo con una app Android que consulta un servicio HTTPS (Hypertext Transfer Protocol Secure)
y recibe una respuesta exitosa. El certificado del servidor cubre el dominio accedido, está
dentro de su período de validez y su cadena llega a una autoridad certificadora aceptada por la
app. Voy a llamar a esa autoridad CA (Certificate Authority).

Además de esas verificaciones, el cliente aplica pinning a la clave pública del certificado
final del servidor, también llamado leaf. Llamo a esa clave **A**.

El pin configurado en la app es un hash de SPKI (Subject Public Key Info), la estructura que
describe la clave pública dentro del certificado. En la implementación usada en este estudio,
ese hash se calcula con SHA-256. La app no guarda la clave privada del servidor: guarda un valor
que le permite reconocer la clave pública esperada.

Eso agrega una condición a la conexión. La cadena puede ser confiable y el hostname correcto,
pero una clave distinta todavía puede ser rechazada por el pinning.

Mientras el servidor presente un certificado con A, las dos condiciones están satisfechas: la
validación TLS normal acepta el certificado, y la verificación adicional encuentra el pin
esperado. La app recibe `HTTP 200`, el código de una respuesta HTTP exitosa.

Hasta aquí, el control está funcionando. Para entender qué puede pasar durante un mantenimiento,
primero necesito separar dos cambios que suelen tratarse como equivalentes: reemplazar el
certificado y reemplazar la clave.

## Pregunta 1: ¿un certificado nuevo necesita una clave nueva?

En el primer artículo, comparo certificados distintos que llevan la misma clave pública. Esa
distinción también importa en una renovación.

El certificado es un documento firmado: contiene la clave pública, los nombres para los que fue
emitido, su período de validez y otros campos. El fingerprint es el hash de ese documento
completo. El pin de SPKI que estoy estudiando corresponde a la estructura de la clave pública.

Si un certificado se reemite preservando el mismo SPKI, su número de serie, validez y firma
pueden cambiar, igual que el fingerprint. El pin de esa clave, sin embargo, sigue coincidiendo.
La conexión seguirá dependiendo de que el nuevo certificado y su cadena pasen las demás
validaciones.

Verifico esa distinción antes de rotar la clave. Primero, el servidor usa el certificado
original con A. Después, presento un certificado reemitido con la misma clave, pero con otro
número de serie y otro período de validez. El cliente sigue aceptando solo el pin de A.

El fingerprint SHA-256 cambia de `92:41:74:CF…` a `17:2D:64:C1…`, mientras el hash de SPKI
permanece igual. En los dos casos, la app recibe `HTTP 200`.

| Evidencia observada | Certificado original | Certificado reemitido |
| --- | --- | --- |
| Fingerprint SHA-256, abreviado | `92:41:74:CF…` | `17:2D:64:C1…` |
| Clave pública | A | A |
| Cliente que acepta solo A | HTTP 200 | HTTP 200 |

Los [valores completos y certificados públicos](#TODO-link-github-evidence-article-04-certificate-comparison)
permiten revisar la comparación. La reemisión cambió el documento, pero preservó la
correspondencia del pin. Por eso, la frase "la renovación rompió el pinning" necesita una
pregunta adicional: **¿también cambió la clave pública?**

| Cambio | Qué pasa con el pin de la clave del servidor |
| --- | --- |
| Certificado reemitido con el mismo SPKI | El hash de SPKI sigue coincidiendo |
| Certificado emitido con otra clave pública | El cliente necesita conocer un pin compatible con esa otra clave |
| Certificado vencido o inválido para el dominio | Un pin que coincide no exime la validación normal del certificado |

Para investigar la segunda situación, reemplazo A por otro par de claves. Llamo a la nueva clave
pública **B**. El servidor recibe un certificado que contiene B y usa la clave privada
correspondiente durante la conexión.

El nuevo certificado sigue siendo válido para el mismo dominio y es emitido por la misma CA. Lo
que necesita cambiar para el pinning es el hash de la clave pública presentada.

## Pregunta 2: ¿por qué la app rechaza un servidor que sigue siendo legítimo?

Después de la rotación, el cliente que solo conoce A deja de recibir la respuesta. La pantalla
muestra una falla, con `Exception: SSLHandshakeException` y causa raíz `CertificateException:
Pin verification failed`.

<img src="/images/posts/04-por-que-backup-pins-e-rotacao-importam-em-producao/profile-b-nscPinnedPrimaryOnly.png" alt="Cliente que solo conoce A mostrando FAILURE en rojo y Pin verification failed después de la rotación legítima a la clave B" class="center" width="400">

*Después de la rotación legítima a B, el cliente que solo conoce A rechaza la conexión porque los pines no coinciden.*

Este resultado recuerda el rechazo del proxy estudiado en el segundo artículo. Pero, esta vez, la
conexión va directamente al servidor legítimo. No hay un intermediario reemplazando el
certificado en el camino.

Verifico los dos certificados: ambos siguen siendo válidos para el hostname y para su uso como
servidor TLS, con una cadena que termina en la CA aceptada por los clientes. La diferencia
relevante para la regla de la app es la clave pública.

El servidor ahora presenta B. El cliente todavía espera A.

- **Antes de la rotación:** cliente espera A → servidor presenta A → conexión aceptada.
- **Después de la rotación:** cliente espera A → servidor presenta B → el pinning rechaza.

La CA afirma que el nuevo certificado es válido para ese dominio. El cliente, por su parte, sigue
aplicando una restricción más específica: solo acepta una cadena que coincida con los pines que
recibió.

Esa regla no puede distinguir la intención de quien hizo el cambio. Una clave inesperada puede
venir de una interceptación o de un mantenimiento legítimo. En ambos casos, falta la
correspondencia esperada.

Por eso configurar el pinning correctamente no basta para garantizar continuidad. Al distribuir
el pin de A, también creo una dependencia entre la app ya instalada y la configuración futura
del servidor.

## Pregunta 3: ¿esa interrupción podría haberse evitado?

Para responder, comparo el resultado con otro cliente. Consulta el mismo servicio y hace las
mismas verificaciones de certificado, pero ya conoce dos alternativas: los pines de A y de B.
Esa configuración estaba instalada antes de la rotación en el servidor.

Mientras el servidor sigue usando A, los dos clientes funcionan. Después de promover B, el
comportamiento se divide:

| Política instalada en el cliente | Servidor con A | Servidor con B |
| --- | --- | --- |
| Acepta solo el pin de A | Conecta | Rechaza por pinning |
| Acepta los pines de A y B | Conecta | Conecta |

En la pregunta anterior, el cliente que solo conoce A rechazó la conexión. La captura de abajo
muestra el cliente que ya conoce A y B, conectado directamente al mismo servidor después de la
misma rotación legítima.

<img src="/images/posts/04-por-que-backup-pins-e-rotacao-importam-em-producao/profile-b-nscPinned.png" alt="Cliente que ya conoce A y B recibiendo HTTP 200 después de la misma rotación de clave en el servidor" class="center" width="400">

*El cliente sigue conectado porque B ya era una alternativa aceptada por la política instalada.*

El segundo cliente sigue recibiendo `HTTP 200`. No necesito quitar la verificación de pines ni
instalar una nueva CA entre los dos estados. B satisface una alternativa que la app ya había
recibido.

Es esa alternativa preparada con anticipación la que llamo **backup pin**.

La palabra "backup" puede sugerir un mecanismo que queda apagado hasta que la clave principal
falla. En la configuración usada aquí, el conjunto de pines funciona como una lista de
alternativas aceptadas. No hay una prioridad que obligue a Android a intentar A antes de permitir
B.

El pin de B ya participa en la decisión desde que la política está en uso. Su papel de backup
viene de la planificación: el servidor todavía usa A, pero el cliente está preparado para
aceptar B. El código de la plataforma aplica esa correspondencia en
[`NetworkSecurityTrustManager.checkPins`](https://github.com/aosp-mirror/platform_frameworks_base/blob/master/core/java/android/security/net/config/NetworkSecurityTrustManager.java).

Tampoco es necesario que el servidor presente las dos claves al mismo tiempo para que este
experimento funcione. Pasa de un certificado con A a otro con B; la política del cliente acepta
las dos posibilidades.

Aquí, los pines corresponden a las claves del leaf. Fijar la clave de una CA cambiaría el alcance
de la restricción: otros certificados con nuevas claves de servidor podrían seguir siendo
compatibles con esa CA. La comparación de este artículo mantiene el pin en la clave del servidor
para hacer explícito el efecto de la rotación.

## Pregunta 4: ¿un segundo hash es suficiente para tener un backup?

El resultado muestra el valor de aceptar B con anticipación. Pero abre otra pregunta: si pongo
cualquier hash adicional en la app, ¿ya tengo una alternativa utilizable?

Para aceptar la conexión, la app necesita encontrar una correspondencia con un pin. Para ofrecer
esa conexión, el servidor necesita algo más: la clave privada de B y un certificado válido que
contenga la clave pública correspondiente.

Si configuro un hash sin tener acceso a la clave privada asociada, no puedo simplemente
convertirlo en una identidad utilizable por el servidor. El mismo problema aparece si tengo la
clave, pero no puedo obtener o instalar su certificado cuando necesito hacer el cambio.

En el experimento, B es un par de claves separado de A. El certificado de B se emite usando la
clave pública que corresponde al pin de backup ya configurado. La comparación de SPKI confirma
esa relación antes de la promoción.

Ese orden importa: si genero otra clave en el momento del cambio, tendrá otro SPKI y no
coincidirá con el backup que distribuí.

Por lo tanto, preparar el backup implica dos responsabilidades relacionadas. La política del
cliente necesita aceptar B, y la operación del servidor necesita poder usar B. Eso incluye
preservar la clave privada de forma adecuada, controlar su acceso y tener un camino viable para
la emisión e instalación del certificado.

El experimento local demuestra la correspondencia entre clave y pin. En un servicio real, la
capacidad de ejecutar esa rotación necesita estar respaldada por la gestión de las claves y los
certificados.

## Pregunta 5: ¿el backup también protege a quien todavía no actualizó la app?

El cliente con A y B sobrevive al cambio porque recibió el pin de B antes de que ocurriera. Esa
observación impone un límite importante: el backup solo ayuda a las instalaciones que ya lo
conocen.

Imagina que una versión antigua se distribuyó con solo A. Después, publico una versión que
acepta A y B. Mientras el servidor sigue en A, ambas logran conectar. Ese período permite
distribuir la nueva configuración sin exigir de inmediato la nueva clave.

Cuando el servidor pasa a B, sin embargo, la versión antigua sigue esperando solo A. Publicar
la actualización no cambia la política de las instalaciones que todavía no la recibieron.

Hay usuarios que postergan actualizaciones, dispositivos que quedan sin conexión y versiones que
dejan de recibir nuevas releases. La decisión de promover B necesita considerar cuáles de esos
clientes siguen siendo soportados. No existe un número universal de días que garantice la
actualización de todos.

Empiezo a ver la rotación como un cambio coordinado entre dos lados: el servidor puede adoptar B
rápidamente, pero el conocimiento de B necesita llegar a una población de clientes con ritmos
distintos de actualización.

Es esa diferencia de ritmo la que convierte al backup pin en algo operacionalmente importante.
Abre una ventana de compatibilidad. Por sí sola, no garantiza que toda la base esté dentro de
ella.

## Pregunta 6: si B se volvió la clave actual, ¿qué pasa en la próxima rotación?

Después de promover B, podría mantener la política A+B indefinidamente. Seguiría aceptando la
clave actual, pero no prepararía una futura clave C.

Para repetir la transición con continuidad, necesito distribuir una política que acepte B y C
mientras el servidor todavía usa B. La misma lógica que permitió la primera rotación pasa a
guiar la siguiente:

| Momento | Clave usada por el servidor | Política que preparo en los clientes |
| --- | --- | --- |
| Antes de la primera rotación | A | Aceptar A y B |
| Después de promover B | B | Distribuir una versión que acepte B y C |
| Después de preparar a los clientes para C | C | Mantener una alternativa para el próximo cambio |

Durante el período en que el servidor usa B, los clientes A+B y B+C tienen una clave aceptada en
común. Eso permite actualizar la configuración sin interrumpir de inmediato las conexiones de
quien todavía está en la versión anterior.

Pero cuando el servidor pasa a C, los clientes que se quedaron en A+B vuelven a enfrentar el
mismo problema. La continuidad depende de distribuir la próxima alternativa antes de exigirla.

También hay una consecuencia para la eliminación de claves antiguas. Cambiar el servidor a B no
quita A de la política A+B ya instalada. El pin antiguo sigue siendo una alternativa reconocida
por esa configuración, sujeto a las demás validaciones del certificado.

Si la motivación de la rotación es un compromiso de A, eso importa. Dejar de usar una clave en
el servidor y dejar de aceptarla en los clientes son cambios distintos. La actualización de la
política, la validación de certificados y los procedimientos de respuesta necesitan
considerarse en conjunto.

Así, la planificación no termina cuando B entra en uso. Necesita incluir la próxima clave y el
destino de las claves que dejaron de ser deseables.

## Pregunta 7: ¿expirar los pines resuelve la incompatibilidad?

Existe otro mecanismo que puede parecer una salida para los clientes que no se actualizan: la
expiración de la política de pinning.

En Android, el Network Security Config (NSC) permite definir una fecha en el atributo
`expiration` del conjunto de pines. Después de esa fecha, esos pines dejan de aplicarse; sin ese
atributo, el conjunto no recibe esa fecha de expiración. La
[referencia del NSC](https://developer.android.com/privacy-and-security/security-config#pin-set)
documenta ese comportamiento.

Esa expiración es distinta de la validez del certificado del servidor. Si el certificado sigue
siendo válido, pero la conexión fallaba solo porque ningún pin coincidía, dejar de aplicar los
pines puede permitir que vuelva a funcionar. Si el certificado está vencido o no representa el
dominio accedido, la expiración de los pines no corrige ese otro problema.

Esto crea una decisión entre continuidad y duración de la restricción adicional. Un cliente que
nunca recibe nuevos pines puede dejar de exigir esa correspondencia después de la fecha
configurada. Al mismo tiempo, pierde la protección adicional que ofrecían los pines.

No trato la expiración como una renovación automática ni como un sustituto para distribuir las
próximas claves. Necesita ser parte de la política de mantenimiento: ¿por cuánto tiempo quiere
el equipo exigir esa restricción en un cliente que se quedó con la configuración antigua?

## Pregunta 8: ¿qué cambia cuando la rotación tiene que ser de emergencia?

En la transición planificada, B ya está preparada y su pin ya fue distribuido. Una emergencia
puede reducir ese margen de elección. La clave de backup puede estar indisponible, la emisión
del certificado puede demorar o el cambio necesario puede exigir otra clave.

El pin adicional no elimina esas dependencias. Para contar con B como recuperación, necesito
saber si puedo ponerla en uso dentro del tiempo necesario y qué clientes lograrán reconocerla.

También necesito pensar en el camino de vuelta. Si una rotación planificada presenta un
problema, volver temporalmente a A puede ser una posibilidad. Si A fue comprometida, ese mismo
regreso puede ir en contra del objetivo del cambio. "Tenemos rollback" solo es una respuesta
útil cuando sé qué estado anterior sigue siendo aceptable.

Otro cuidado aparece cuando considero actualizar los pines remotamente. Si la app necesita
buscar la nueva configuración por el mismo endpoint cuya clave está rechazando, puede quedarse
sin el canal necesario para recibir la corrección. Una solución de configuración remota
exigiría un proyecto propio de autenticidad, integridad y recuperación.

Quitar la validación de cadena o de hostname para restablecer la comunicación también cambia el
problema: la conexión pierde garantías básicas de autenticación del servidor. La recuperación
necesita preservar esas garantías y tener un camino que los clientes afectados realmente puedan
usar.

## ¿Cómo notaría esta falla en una app distribuida?

En el experimento, la excepción es visible. Para un usuario, el mismo evento puede aparecer solo
como "no se pudo conectar". La API puede seguir saludable y atendiendo a otros clientes, mientras
una versión específica de la app empieza a rechazar su certificado.

Por eso, mirar solo la disponibilidad del servidor no basta para interpretar el incidente.
Necesito relacionar la versión del cliente, la política que recibió y la clave que realmente
presentó el punto de terminación TLS al que se conectó.

Un cambio aplicado gradualmente en varios servidores también puede producir fallas
intermitentes: el cliente acepta la clave servida por uno de ellos y rechaza la de otro. Para
investigar, necesito saber si todos los caminos de atención están presentando certificados
compatibles con las versiones todavía soportadas.

El canal de telemetría merece la misma atención. Si los eventos de error dependen de la conexión
que acaba de fallar, la ausencia de registros no demuestra que los clientes estén funcionando.
Puede ser justamente parte de la pérdida de conectividad que necesito detectar.

Esas consecuencias amplían lo que significa operar el control: saber qué claves deberían ser
aceptadas, seguir la distribución de esa política y lograr reconocer cuándo cliente y servidor
dejaron de ser compatibles.

## Después de esta investigación, ¿cuándo vale la pena adoptar pinning?

El experimento muestra que una rotación legítima puede ser aceptada por un cliente preparado y
rechazada por otro que solo conoce la clave anterior. La ganancia de restringir la confianza
viene acompañada de la necesidad de mantener esa configuración compatible con el servicio.

Ese costo ayuda a entender la advertencia de la documentación de Android, que
[no recomienda certificate pinning para apps Android](https://developer.android.com/privacy-and-security/security-ssl#restricting-your-app-to-specific-certificates)
por el riesgo de que los cambios en el servidor dejen a los clientes sin conectividad. Cuando se
usa la restricción, el documento destaca los pines de backup y la expiración entre los cuidados
operacionales.

Este riesgo no es exclusivo de Android: la web ya pasó por él a escala real. El HTTP Public Key
Pinning (HPKP), estandarizado en la [RFC 7469](https://www.rfc-editor.org/rfc/rfc7469), fue
descontinuado por los principales navegadores alrededor de 2018 después de que operadores se
bloquearan a sí mismos el acceso a sus propios dominios al configurar pines sin una clave de
backup viable. El experimento de este artículo reproduce, a escala de laboratorio, el mismo
mecanismo de falla que llevó al abandono de ese estándar.

Para evaluar una adopción, necesito partir de la amenaza concreta que pretendo dificultar.
Después, necesito verificar si controlo las claves, los certificados, la terminación TLS y la
distribución necesaria para mantener funcionando las versiones soportadas.

El tercer artículo muestra el límite frente a alguien que controla el proceso o altera el
paquete de la app. Este último agrega la responsabilidad de mantener el control funcionando
durante cambios legítimos. Las dos perspectivas necesitan participar de la misma decisión.

La autenticación, la autorización y las validaciones en el backend siguen siendo necesarias.
Una restricción adicional en el canal no convierte al cliente en un entorno confiable ni
sustituye esos controles.

## Para quien quiera reproducir la investigación

La demostración A → B se ejecutó con dos clientes Android y certificados emitidos por una CA
local. La [guía opcional de reproducción](#TODO-link-github-reproducao-rotation)
reúne los comandos, los nombres de las variantes, la preparación y los resultados esperados.
Ninguno de estos pasos es necesario para seguir la explicación de este artículo.

Las [nuevas evidencias](#TODO-link-github-evidence-article-04-readme) incluyen las capturas, los
certificados públicos y los registros de la reemisión con la misma clave A y de la rotación A →
B. Las discusiones sobre C, emergencias, expiración y observabilidad desarrollan sus
consecuencias operacionales, sin representar pruebas adicionales de producción.

## Lo que cierra la serie

Empiezo la serie investigando cómo un cliente decide confiar en el certificado de un dominio.
Después, separo la elección de las CAs de la restricción adicional de los pines y sigo los
puntos en los que esa restricción puede aplicarse y alterarse.

En esta última investigación, la pregunta pasa a ser cómo mantener esa confianza durante un
cambio legítimo del servicio. El cliente que conoce A y B sigue conectado porque la alternativa
llegó antes de ser necesaria. El cliente que solo conoce A pierde la conexión, aunque el
servidor sigue presentando un certificado válido.

El modelo mental con el que termino es:

- *El certificado puede cambiar sin que la clave cambie.*
- *Si la clave cambia, el cliente necesita reconocer una alternativa compatible.*
- *El backup solo ayuda a quien recibió ese pin antes de la rotación.*
- *La próxima alternativa necesita llegar antes de que el servidor la exija.*

El pin distribuido hoy pasa a formar parte de las condiciones para cambiar el servicio mañana.
Justificar la adopción de pinning exige entender la amenaza que dificulta, el atacante que
todavía logra alterarlo y el mantenimiento necesario para que sus propios usuarios sigan
conectados.

## Referencia rápida de los términos usados

| Término | Significado en este artículo |
| --- | --- |
| TLS (Transport Layer Security) | Protocolo usado para proteger y autenticar la conexión |
| CA (Certificate Authority) | Autoridad certificadora aceptada en la validación de la cadena |
| Leaf | Certificado final presentado para representar al servidor |
| SPKI (Subject Public Key Info) | Estructura de la clave pública cuyo hash se usa en los pines |
| Fingerprint | Hash del certificado completo |
| NSC (Network Security Config) | Mecanismo declarativo de configuración de red de Android |
| Backup pin | Pin de una clave alternativa preparada antes de una rotación |
| Rotación A → B | Reemplazo de la clave A por la clave B en el servidor |
| Expiration | Fecha después de la cual el NSC deja de aplicar ese conjunto de pines |
| Rollback | Retorno a un estado anterior mientras siga siendo aceptable |
