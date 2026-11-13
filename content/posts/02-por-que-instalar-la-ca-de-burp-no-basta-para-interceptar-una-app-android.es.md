+++
title = "Por qué instalar la CA de Burp no basta para interceptar una app Android"
translationKey = "android-burp-ca-trust-vs-pinning"
date = "2026-11-09"
author = "Thiago Siqueira"
tags = ["android", "ssl-pinning", "mobile", "burp", "network-security-config"]
keywords = ["android", "network security config", "certificate pinning", "burp suite", "trust store"]
description = "Un laboratorio Android con múltiples flavors para separar, paso a paso, un fallo común de trust store de un fallo real de certificate pinning vía Network Security Config, usando Burp Suite como proxy MITM."
showFullContent = false
readingTime = true
hideComments = false
+++

Ya tengo la CA (Certificate Authority) del Burp instalada en el Android.

El Chrome consigue abrir la API (Application Programming Interface) del laboratorio.

El Burp consigue interceptar la conexión HTTPS (Hypertext Transfer Protocol Secure) y mostrar la solicitud en su historial.

Entonces, ¿por qué una app Android sigue fallando?

Esa es la pregunta que abre la segunda parte de mi laboratorio de SSL pinning.

En el artículo anterior, quise entender cómo un cliente valida la autenticidad de un dominio durante el handshake TLS (Transport Layer Security). Recorrí leaf, intermediate, trust anchor, SAN (Subject Alternative Name), SPKI (Subject Public Key Info) y cross-signing hasta llegar a un modelo mental importante: *una conexión TLS solo se acepta cuando el cliente consigue construir un camino hasta una trust anchor en la que ya confía localmente.*

Ese modelo ya responde en qué CAs confía la app. Pero hay una pregunta adicional que no cubre: después de confiar en la CA, ¿la app confía en cualquier clave pública que esa CA firme? Esa es la pregunta que le da nombre a certificate pinning, también llamado SSL pinning: la técnica por la cual una aplicación, además de validar la cadena hasta una CA confiable, también exige que la clave pública presentada por el servidor coincida con un conjunto específico de claves esperadas, los pines. Esa es la técnica que quiero investigar en el resto de la serie, y este segundo artículo empieza aislando la parte que más se confunde: separar un fallo común de trust store de un fallo real de pinning.

Para responder esto sin depender de un servicio real, armé un laboratorio pequeño y controlado.

## Antes de empezar: el laboratorio del experimento

No quería estudiar pinning contra una API real, porque eso mezclaría mi experimento con certificados, claves, rotación y disponibilidad de un servicio que no controlo.

Entonces construí una API HTTPS local, pequeña y descartable. Existe solo para servir como blanco previsible del laboratorio:

- controlo el hostname;
- controlo el certificado leaf, es decir, el certificado final presentado por el servidor;
- controlo la Root CA que firma ese certificado;
- puedo calcular exactamente el SPKI, es decir, la estructura de la clave pública cuyo hash la app debe aceptar;
- puedo cambiar la clave después, cuando quiera estudiar rotación.

Expone un endpoint `/health`, usado para validar conectividad y TLS, y un endpoint `/profile`, consultado por la app para recibir un payload ficticio.

También creé una app Android mínima. Tiene una única pantalla, un botón para disparar la solicitud y un área donde puedo ver `SUCCESS`, `FAILURE`, el estado HTTP o la excepción capturada. No intenta simular una aplicación de negocio real; existe para hacer visible en qué etapa la conexión se acepta o se rechaza.

El punto más importante es que no creé varias apps con lógicas distintas. Creé la misma app con varios flavors, es decir, variantes de build. Todas usan la misma `MainActivity`, llaman a la misma URL `/profile` y muestran la misma interfaz. Lo que cambia entre ellas es solo la política de confianza. Esto fue importante porque me permitió comparar una variable a la vez. Si dos flavors llaman a la misma API, por el mismo camino, y solo una falla, la diferencia no está en el backend ni en la red: está en la política de confianza de esa variante.

La arquitectura del laboratorio quedó así: **Android Emulator → (HTTPS) → Burp Suite en macOS → (HTTPS) → API local en 127.0.0.1:8443**.

El Android Emulator representa el device analizado. El Burp Suite corre en macOS y queda en medio de la conexión como proxy MITM (Man-in-the-Middle). La API local también corre en macOS, pero en otro puerto, sirviendo el certificado y el payload que controlo.

El endpoint de conectividad estaba en `https://api.ssl-pinning.127.0.0.1.nip.io:8443/health`.

En la práctica, alternaba entre dos caminos:

- **Android Emulator → API local**
- **Android Emulator → Burp Suite → API local**

## La primera evidencia

Con el laboratorio armado, quiero reproducir una situación común en análisis mobile:

1. configurar el Burp Suite como proxy;
2. instalar la CA del Burp en el Android;
3. confirmar que el navegador acepta la conexión;
4. abrir una app Android;
5. observar que todavía falla;
6. descubrir si el fallo viene de trust store o de certificate pinning.

El hilo conductor del artículo pasa a ser:

- *¿Qué está rechazando la app?*
- *¿Una CA en la que no confía?*
- *¿O una clave pública que no coincide con el pin esperado?*

Antes de probar cualquier app, valido los dos caminos en el Chrome del Android.

Sin proxy:

```bash
adb shell settings put global http_proxy :0
adb shell am start -a android.intent.action.VIEW \
  -d 'https://api.ssl-pinning.127.0.0.1.nip.io:8443/health?mode=android-direct'
```

Con proxy:

```bash
adb shell settings put global http_proxy 10.0.2.2:8080
adb shell am start -a android.intent.action.VIEW \
  -d 'https://api.ssl-pinning.127.0.0.1.nip.io:8443/health?mode=android-burp'
```

Los comandos anteriores hacen dos cosas distintas:

| Fragmento | Función |
| --- | --- |
| `settings put global http_proxy :0` | Quita el proxy global del Android |
| `settings put global http_proxy 10.0.2.2:8080` | Hace que el emulador envíe tráfico al Burp en el host |
| `10.0.2.2` | Alias especial que, dentro del emulador, apunta al host macOS |
| `am start -a android.intent.action.VIEW -d <url>` | Le pide al Android que abra la URL en el navegador |

Uso el hostname `127.0.0.1.nip.io` porque necesito que el mismo nombre DNS funcione tanto en el emulador como en el Burp, manteniendo la validación de hostname del TLS.

En el Android, el puente hasta la API local depende de un puerto redirigido desde el host: `adb reverse tcp:8443 tcp:8443` redirige el puerto 8443 del emulador al mismo puerto en macOS, donde la API local está escuchando.

En los dos casos, el Chrome muestra el JSON de salud de la API:

```json
{
  "status": "ok",
  "service": "ssl-pinning-lab-api",
  "time": "..."
}
```

En el segundo caso, la solicitud también aparece en el HTTP history del Burp.

<img src="/images/posts/02-por-que-instalar-a-ca-do-burp-nao-basta-para-interceptar-um-app-android/chrome-and-burp-health.png" alt="Chrome abriendo el endpoint health mientras el Burp registra la misma solicitud" class="center" width="820">

*El Chrome acepta la conexión interceptada y la solicitud aparece en el historial del Burp.*

Para interpretar esta prueba, hay un detalle importante: en el conjunto de certificados instalados por el usuario en el Android, al que más adelante voy a llamar user store, ya había instalado dos CAs distintas:

- la Root CA del laboratorio, que firma el certificado servido directamente por la API;
- la CA del Burp, que firma los certificados generados dinámicamente durante el MITM.

Cuando accedo a la API directamente, el Android necesita confiar en la Root CA del laboratorio. Cuando paso por el Burp, el Android deja de ver el certificado original de la API y empieza a ver un certificado emitido por la CA del Burp.

Esto prueba cinco cosas:

1. la API es accesible;
2. el hostname es correcto;
3. el Chrome acepta la Root CA del laboratorio en el camino directo;
4. el Chrome acepta la CA del Burp en el camino interceptado;
5. el proxy consigue interceptar el HTTPS del Chrome en el device.

Pero esto todavía solo prueba el comportamiento del Chrome y de la infraestructura. La app Android todavía puede tomar una decisión de confianza distinta.

Entonces paso a la app.

## Antes de probar los flavors: ¿dónde entra la política de confianza?

Como todos los flavors usan la misma `MainActivity` y llaman a la misma URL `/profile`, necesito encontrar dónde entra realmente la política de confianza en la app.

Antes de correr la primera variante, busco esa declaración en el `AndroidManifest.xml`. Ahí encuentro la referencia a un archivo llamado `network_security_config`:

```xml
<application
    android:networkSecurityConfig="@xml/network_security_config"
    ...>
```

Network Security Config, o NSC, es el mecanismo declarativo del Android para definir trust anchors, reglas de cleartext, pines y excepciones de debug.

El manifest es común a todos los flavors. Lo que cambia en el build es el recurso que ocupa ese camino: `app/src/<flavor>/res/xml/network_security_config.xml`.

Es decir, el código Java sigue abriendo la misma URL, pero el Android empieza a consultar una política de confianza distinta según el flavor instalado.

El Chrome usa `/health` solo para validar la infraestructura. A partir de aquí, los flavors de la app llaman a `/profile`, siempre por el mismo camino, cambiando solo el XML de confianza.

Esto define la comparación que hago desde aquí en adelante:

- misma pantalla;
- misma URL;
- mismo Burp;
- misma API;
- **XML de confianza distinto**.

## Pregunta 1: ¿qué pasa con un flavor que solo confía en CAs del sistema?

Empiezo por la variante `systemOnly`.

Usa `HttpsURLConnection`, el cliente HTTPS nativo de la plataforma Java/Android, y una configuración de red mínima:

```xml
<network-security-config>
    <base-config cleartextTrafficPermitted="false">
        <trust-anchors>
            <certificates src="system" />
        </trust-anchors>
    </base-config>
</network-security-config>
```

El nombre ya da una pista: este flavor solo confía en CAs del sistema.

La ejecuto con el Burp todavía configurado como proxy:

```bash
adb shell am force-stop com.example.sslpinninglab.systemonly
adb shell am start \
  -n com.example.sslpinninglab.systemonly/com.example.sslpinninglab.MainActivity \
  --ez auto_request true
```

Este patrón de comando se repite en los próximos flavors:

| Fragmento | Función |
| --- | --- |
| `am force-stop <package>` | Termina la instancia anterior para que la prueba empiece limpia |
| `am start -n <package>/<activity>` | Abre exactamente el flavor deseado |
| `--ez auto_request true` | Envía un boolean extra para que la app dispare la solicitud al abrirse |

La pantalla muestra `FAILURE`, con `Exception: SSLHandshakeException` y causa raíz `CertPathValidatorException: Trust anchor for certification path not found.`

<img src="/images/posts/02-por-que-instalar-a-ca-do-burp-nao-basta-para-interceptar-um-app-android/system-only-burp-failure.png" alt="Flavor systemOnly fallando vía Burp con CertPathValidatorException" class="center" width="420">

*`systemOnly` falla vía Burp antes de cualquier comparación de pines.*

El logcat, es decir, el stream de logs del Android, confirma la misma causa: `Caused by: java.security.cert.CertPathValidatorException: Trust anchor for certification path not found.`

En este momento, aparece la primera conclusión importante: *ese fallo todavía no es pinning.*

La app ni siquiera llega a comparar SPKIs.

Falla antes, porque la cadena presentada por el Burp no termina en una CA aceptada por esa variante.

De hecho, `systemOnly` también falla sin el Burp. La Root CA de la propia API local también está instalada en el user store, y este flavor no acepta ninguna CA proveniente de ahí. Esto refuerza el punto: `systemOnly` no existe para probar pinning; existe para probar que una app puede rechazar la conexión incluso antes de que exista cualquier pin.

## Pregunta 2: ¿por qué la CA del Burp instalada en el Android no entra en la validación?

La CA del Burp está instalada en el Android, pero vive en el user store, es decir, en el conjunto de certificados agregados por el usuario.

Este detalle importa porque las apps Android modernas no confían en user-added CAs por defecto.

En apps con `targetSdkVersion >= 24`, es decir, apps que declaran Android 7.0 o superior como versión objetivo, el comportamiento por defecto pasa a ser confiar en CAs del sistema y no confiar automáticamente en CAs agregadas por el usuario.

En el laboratorio, `systemOnly` hace explícita esa decisión con `src="system"`. En cambio, `proxyTrusted` hace el opt-in contrario, agregando `src="user"` de forma deliberada.

Esto explica la aparente paradoja:

| Cliente | Qué acepta | Resultado |
| --- | --- | --- |
| Chrome Android | CA instalada por el usuario | Pasa |
| `systemOnly` | Solo CAs del sistema | Falla |

La pregunta correcta no es *¿la CA del Burp está instalada?* La pregunta correcta es *¿esta app decidió confiar en CAs agregadas por el usuario?*

En el caso de `systemOnly`, la respuesta es no.

Este es un punto importante del laboratorio porque evita un error común: *ver un fallo de handshake y concluir de inmediato que existe pinning.*

En este caso, el fallo viene solo de la política de trust anchors.

## Pregunta 3: ¿qué cambia cuando la app empieza a aceptar user-added CAs?

Para responder esto, corro la variante `proxyTrusted`.

Llama al mismo endpoint, usa el mismo Burp y mantiene el mismo flujo de red. El único cambio está en el Network Security Config:

```xml
<network-security-config>
    <base-config cleartextTrafficPermitted="false">
        <trust-anchors>
            <certificates src="system" />
            <certificates src="user" overridePins="false" />
        </trust-anchors>
    </base-config>
</network-security-config>
```

Ahora la app acepta:

- CAs del sistema;
- CAs agregadas por el usuario.

El atributo `overridePins="false"` hace explícita una decisión que ya sería el valor por defecto fuera de `debug-overrides`. Dice que una CA proveniente del user store puede participar en la validación normal de la cadena, pero no gana permiso para ignorar pines si existen. En `proxyTrusted`, todavía no hay `<pin-set>`, así que este detalle no cambia el resultado. Documenta la intención que voy a mantener en la comparación con `nscPinned`; omitir el atributo aquí no cambiaría su valor por defecto.

Ejecuto:

```bash
adb shell am force-stop com.example.sslpinninglab.proxytrusted
adb shell am start \
  -n com.example.sslpinninglab.proxytrusted/com.example.sslpinninglab.MainActivity \
  --ez auto_request true
```

El resultado es `SUCCESS`, con `HTTP 200`.

En la pantalla, la app también muestra el payload devuelto por la API:

```json
{
  "user": "alice.demo",
  "accountTier": "lab",
  "message": "Fictional data for SSL pinning study only"
}
```

<img src="/images/posts/02-por-que-instalar-a-ca-do-burp-nao-basta-para-interceptar-um-app-android/proxy-trusted-burp-success.png" alt="Flavor proxyTrusted funcionando vía Burp y devolviendo HTTP 200" class="center" width="420">

*`proxyTrusted` confía en la CA del Burp y consigue leer el payload de la API.*

El logcat también registra: `proxy-trusted request succeeded`.

Esta variante es la baseline que necesito.

Prueba que:

1. el Burp está funcionando;
2. la CA del Burp es confiable para esa app;
3. la ruta hasta la API es correcta;
4. el handshake TLS puede terminar con éxito;
5. cualquier fallo posterior en otra variante ya no puede atribuirse simplemente al proxy.

También pasa en el camino directo, porque `src="user"` acepta la Root CA local de la API. En otras palabras, `proxyTrusted` es el primer flavor en el que tanto el camino directo como el camino vía Burp funcionan.

Ese es el papel más importante de `proxyTrusted` en el laboratorio.

No existe para mostrar una app "vulnerable".

Existe para probar que la infraestructura de interceptación es correcta antes de hablar de pinning.

## Pregunta 4: ¿qué pasa cuando agrego un `<pin-set>` a la misma base de confianza?

Después de validar `proxyTrusted`, corro la variante `nscPinned` para responder exactamente esa pregunta.

Usa exactamente la misma base de confianza:

```xml
<trust-anchors>
    <certificates src="system" />
    <certificates src="user" overridePins="false" />
</trust-anchors>
```

Pero agrega un `pin-set` para el dominio de la API:

```xml
<domain-config cleartextTrafficPermitted="false">
    <domain includeSubdomains="false">api.ssl-pinning.127.0.0.1.nip.io</domain>
    <domain includeSubdomains="false">api.ssl-pinning.10.0.2.2.nip.io</domain>
    <pin-set expiration="2035-01-01">
        <pin digest="SHA-256">Kyfpm+U4uD7FC1lNrw/7LHwerRcQ0EIMmaWCZd0dk8Q=</pin>
        <pin digest="SHA-256">jN72Sgrbs/bv6VCez+cHlzrPx3iW26rCyaTwmAJB0Tk=</pin>
    </pin-set>
</domain-config>
```

Estos pines no hay que memorizarlos ni inventarlos a mano. Son el mismo hash del SPKI que ya aprendí a extraer en el artículo anterior, solo que en otro formato. Ahí usé:

```bash
openssl x509 -in cert.pem -pubkey -noout \
  | openssl pkey -pubin -outform DER \
  | openssl dgst -sha256
```

para obtener el hash del SPKI en hexadecimal. El Network Security Config espera ese mismo hash en base64, así que el comando gana un paso extra:

```bash
openssl x509 -in cert.pem -pubkey -noout \
  | openssl pkey -pubin -outform DER \
  | openssl dgst -sha256 -binary \
  | openssl base64
```

La diferencia es solo la codificación: `-binary` evita la salida en hexadecimal, y `openssl base64` convierte esos bytes al formato que espera el atributo `<pin digest="SHA-256">`. El concepto sigue siendo el mismo: es el hash del SPKI, no del certificado entero, el que entra en el pin.

Los dos valores del `<pin-set>` no significan que se estén usando dos claves al mismo tiempo en la API. El primero es el pin de la clave actual del servidor. El segundo es un backup pin, reservado para una futura rotación. En este artículo, el punto importante es solo que el certificado generado por el Burp no coincide con ninguno de los dos.

En el camino directo, sin proxy, `nscPinned` pasa. La API local presenta exactamente la clave cuyo hash del SPKI está en el XML. La prueba que quiero observar es el camino vía Burp, donde la cadena todavía puede ser confiable, pero la clave pública presentada cambia.

Primero quito el proxy global del Android y ejecuto el flavor directamente contra la API:

```bash
adb shell settings put global http_proxy :0
adb shell am force-stop com.example.sslpinninglab.nscpinned
adb shell am start \
  -n com.example.sslpinninglab.nscpinned/com.example.sslpinninglab.MainActivity \
  --ez auto_request true
```

El resultado directo es `SUCCESS`, con `HTTP 200`.

Después vuelvo a poner el Burp como proxy y repito exactamente el mismo flavor:

```bash
adb shell settings put global http_proxy 10.0.2.2:8080
adb shell am force-stop com.example.sslpinninglab.nscpinned
adb shell am start \
  -n com.example.sslpinninglab.nscpinned/com.example.sslpinninglab.MainActivity \
  --ez auto_request true
```

El resultado vía Burp es `FAILURE`, con `Exception: SSLHandshakeException` y causa raíz `CertificateException: Pin verification failed`.

El logcat muestra el punto exacto del fallo: `Caused by: java.security.cert.CertificateException: Pin verification failed`, en `NetworkSecurityTrustManager.checkPins(...)`.

Ahora la situación es distinta a `systemOnly`.

En `systemOnly`, la cadena del Burp ni siquiera llega a una trust anchor aceptada.

En `nscPinned`, la CA del Burp ya es aceptada.

El problema ahora es otro: *la clave pública presentada por el Burp no coincide con los SPKIs esperados por la app.*

Este es el punto donde el pinning realmente entra.

Sin pinning, la pregunta de la app es: *¿esta cadena termina en una CA en la que confío?*

Con pinning, la pregunta pasa a ser: *¿esta cadena termina en una CA en la que confío y además contiene una clave pública esperada?*

La comparación central del laboratorio queda:

| Variante | ¿CA del usuario aceptada? | ¿Pinning? | Directo | Vía Burp | Causa del fallo vía Burp |
| --- | --- | --- | --- | --- | --- |
| `systemOnly` | No | No | Falla | Falla | Trust anchor no aceptada |
| `proxyTrusted` | Sí | No | Pasa | Pasa | No hay fallo |
| `nscPinned` | Sí | Sí | Pasa | Falla | El SPKI no coincide |

Esa tabla resume la distinción que quiero probar.

<img src="/images/posts/02-por-que-instalar-a-ca-do-burp-nao-basta-para-interceptar-um-app-android/nsc-pinned-direct-vs-burp.png" alt="A la izquierda, el flavor nscPinned pasa en la solicitud directa a la API; a la derecha, falla cuando la solicitud pasa por el Burp" class="center" width="820">

*A la izquierda, `nscPinned` pasa en la solicitud directa a la API. A la derecha, el mismo flavor falla cuando la solicitud pasa por el Burp y el SPKI presentado cambia.*

## Pregunta 5: ¿en qué momento del flujo ocurre cada fallo?

Los dos fallos aparecen como problemas de TLS para la app, pero no significan lo mismo.

En el escenario con proxy explícito, la app primero establece la conexión TCP (Transmission Control Protocol) con el Burp y solicita un túnel con el método HTTP (Hypertext Transfer Protocol) `CONNECT`. Después, negocia TLS por ese túnel. Entonces, ya existe HTTP de control del proxy antes del TLS; lo que todavía no se envió es la solicitud de la aplicación, como `GET /profile`.

Para los flavors que usan `HttpsURLConnection` y los pines del NSC en este experimento, el flujo es: **TCP con el proxy → CONNECT → handshake TLS → validación de la cadena y de los pines del NSC → solicitud HTTP de la aplicación dentro del túnel TLS.**

El punto relevante para el diagnóstico es que la comunicación TCP con el proxy y el túnel pueden funcionar mientras la validación TLS rechaza la conexión. Eso impide el envío de `GET /profile`, sin que signifique que toda la comunicación anterior falló.

En `systemOnly`, el fallo ocurre cuando el Android intenta encontrar una trust anchor: **TCP OK → empieza el handshake TLS → la cadena del Burp no llega a una CA aceptada → el handshake aborta → `GET /profile` nunca se envía por el túnel TLS.**

En `nscPinned`, la cadena ya es confiable, pero el Android aplica la verificación extra de pines: **TCP OK → empieza el handshake TLS → la cadena del Burp llega a una CA aceptada → `NetworkSecurityTrustManager.checkPins` rechaza el SPKI → el handshake aborta → `GET /profile` nunca se envía por el túnel TLS.**

Esta diferencia importa porque los síntomas visuales pueden parecer iguales: las dos aparecen como `FAILURE` con `SSLHandshakeException`. Pero el diagnóstico correcto depende de la causa raíz:

- en este caso, `CertPathValidatorException` con el mensaje `Trust anchor for certification path not found` → ausencia de una trust anchor aceptada;
- `Pin verification failed` → fallo real de pinning.

## Pregunta 6: ¿un build debug con `<pin-set>` también rechaza al Burp?

Después de separar trust store de pinning, quiero probar una situación que suele confundir los análisis Android: ¿qué pasa cuando un build debug contiene pines, pero también declara una CA específica en `debug-overrides`?

La variante `debugOverrides` tiene un `pin-set`:

```xml
<pin-set expiration="2035-01-01">
    <pin digest="SHA-256">Kyfpm+U4uD7FC1lNrw/7LHwerRcQ0EIMmaWCZd0dk8Q=</pin>
    <pin digest="SHA-256">jN72Sgrbs/bv6VCez+cHlzrPx3iW26rCyaTwmAJB0Tk=</pin>
</pin-set>
```

Pero también tiene:

```xml
<debug-overrides>
    <trust-anchors>
        <certificates src="@raw/burp_ca" overridePins="true" />
    </trust-anchors>
</debug-overrides>
```

Ejecuto:

```bash
adb shell am force-stop com.example.sslpinninglab.debugoverrides
adb shell am start \
  -n com.example.sslpinninglab.debugoverrides/com.example.sslpinninglab.MainActivity \
  --ez auto_request true
```

El resultado es `SUCCESS`, con `HTTP 200`.

<img src="/images/posts/02-por-que-instalar-a-ca-do-burp-nao-basta-para-interceptar-um-app-android/debug-overrides-burp-success.png" alt="Flavor debugOverrides funcionando vía Burp a pesar del pin-set" class="center" width="420">

*`debugOverrides` sigue pasando vía Burp porque la CA de debug puede ignorar los pines.*

El logcat confirma: `debug-overrides request succeeded`.

Esto no significa que el pinning "acepte" el certificado del Burp.

Lo que ocurre es:

1. el build es debuggable, es decir, un build marcado para depuración;
2. la CA del Burp está declarada en `debug-overrides`;
3. la conexión usa esa CA de debug;
4. `overridePins="true"` le indica al Android que ignore los pines para esa anchor.

Esta dispensa de pines es la del NSC. No garantiza que una verificación adicional implementada en una biblioteca o en código propio también sea ignorada.

Vale una salvedad: este comportamiento depende del atributo `android:debuggable` del build, no del nombre del flavor en sí. Cualquier variante compilada como debuggable que declare `debug-overrides` se comportaría de la misma forma, sin importar qué flavor represente. En el laboratorio, `debugOverrides` solo aísla esa configuración para que sea fácil de observar.

En otras palabras: *el pinning no pierde; se salta deliberadamente en la configuración de debug.*

Este detalle importa porque un análisis hecho solo en build debug puede llevar a una conclusión errónea: *"la app acepta al Burp, entonces no tiene pinning"*.

En realidad, la release puede seguir protegida, porque `debug-overrides` solo participa en la política cuando la aplicación es debuggable.

## Lo que el experimento realmente me enseñó

La diferencia entre los flavors hace observable el diagnóstico: `systemOnly` rechaza la cadena, `proxyTrusted` la acepta, y `nscPinned` agrega una restricción que rechaza los pines del proxy. `debugOverrides` muestra una excepción deliberada a la aplicación de los pines del NSC en un build de debug.

Entonces, el mismo resultado visual, `FAILURE`, puede esconder decisiones distintas. La causa raíz, la configuración y la comparación con la baseline son lo que me permite distinguir esas decisiones.

## Nota para quien vaya a implementar pinning

El laboratorio hasta aquí mira por la lente de quien analiza la app. Pero quien configure pinning en una app real va a tomar algunas de estas mismas decisiones desde el otro lado:

- `debug-overrides` con `overridePins="true"` es útil para desarrollo, pero solo es seguro si la flag `android:debuggable` realmente nunca se filtra al build de release;
- en `src="user"`, explicitar `overridePins="false"` documenta la intención, pero eso ya es el valor por defecto fuera de `debug-overrides`. Dentro de `debug-overrides`, el valor por defecto es `true`. El bypass de los pines del NSC ocurre cuando la cadena usa una fuente configurada con `overridePins="true"`;
- un `<pin-set>` con un único pin es un riesgo operacional: si la clave privada del servidor necesita rotar sin aviso, la app deja de funcionar. El backup pin existe para cubrir exactamente ese escenario.

Estos puntos van a ganar más peso práctico cuando la serie llegue a instrumentación y bypass de pinning, pero ya valen como checklist para quien vaya a revisar la configuración de una app antes de eso.

## Materiales y referencias para reproducir esto

Los experimentos dependen de la API y de los flavors de este laboratorio. Los archivos de apoyo incluyen la [API HTTPS](#TODO-link-github-lab-api-readme), la [app Android](#TODO-link-github-lab-android-app-readme), la [preparación del entorno](#TODO-link-github-readme-environment-setup) y el [guión completo de reproducción](#TODO-link-github-readme-full-walkthrough).

Para armar una copia propia, generar claves nuevas no basta: los pines de los XML tienen que acompañar el SPKI de tus claves, y la CA del Burp declarada en `debugOverrides` tiene que ser la de tu proxy. Los hashes y las capturas de este texto pertenecen a la ejecución documentada; no son valores que cualquier instalación nueva deba copiar sin verificar. En los flavors de este artículo, cambio los pines declarados en `<pin-set>` según las claves generadas y reconstruyo los APKs antes de comparar los resultados.

Para revisar el comportamiento de la plataforma:

- [Android — Network Security Configuration](https://developer.android.com/privacy-and-security/security-config): anchors, pines y configuraciones de debug.
- [Android — atributo overridePins](https://developer.android.com/privacy-and-security/security-config#certificates): valores por defecto y dispensa de pines por fuente de CAs.
- [Android — cambios en Android 7.0](https://developer.android.com/about/versions/nougat/android-7.0-changes#tls-ssl): confianza por defecto en CAs agregadas por el usuario.
- [Android Emulator — red](https://developer.android.com/studio/run/emulator-networking): qué significa `10.0.2.2` en el emulador.

## Modelo mental final

Después de recorrer el experimento, separo estas responsabilidades. Es un modelo conceptual para este laboratorio, sin imponer un orden universal de ejecución entre hostname, cadena y pines:

1. primero la app decide en qué CAs confiar;
2. después el Android valida hostname, vigencia y cadena;
3. si la cadena llega a una trust anchor aceptada, la conexión todavía puede enfrentar una verificación extra;
4. el pinning compara la clave pública presentada con los SPKIs esperados;
5. un fallo antes de esa etapa no es pinning.

O, de forma todavía más corta: *trust store responde "¿en qué CAs confío?"; pinning responde "¿esta clave pública es una de las esperadas?"*

Para atribuir este fallo al pinning, necesito separar el rechazo de la cadena del rechazo de los pines. La baseline es lo que me permite aislar esa diferencia en este experimento.

Esto no significa que una app incapaz de aceptar la CA del proxy no pueda contener pinning. Puede tener los dos controles, con el primer rechazo impidiendo que la ejecución llegue a la verificación de los pines. Presencia de pinning y causa del fallo observado son preguntas distintas.

Hasta aquí, solo probé que el pinning bloquea un MITM de red mientras la app sigue ejecutando su propia política de confianza intacta. Todavía no probé qué pasa cuando alguien controla el proceso de la app o altera el APK.

Después de probar que `nscPinned` falla por pinning real, la próxima pregunta deja de ser "¿por qué falla la conexión?" y pasa a ser: *¿dónde vive esa decisión dentro del APK (Android Package)?*

Esa es la investigación que quiero abrir en el próximo artículo.

## Referencia rápida de los términos usados

| Término | Significado |
| --- | --- |
| Android Emulator | Emulador usado para reproducir el device Android |
| Burp Suite | Proxy de interceptación usado para observar tráfico HTTPS |
| MITM (Man-in-the-Middle) | Intermediación de la conexión entre cliente y servidor |
| CA (Certificate Authority) | Autoridad certificadora que firma certificados |
| Trust store | Conjunto local de CAs confiables disponibles para el cliente |
| Trust anchor | Certificado o clave confiable que termina una validación concreta |
| System store | Conjunto de CAs confiadas por el sistema Android |
| User store | Conjunto de CAs agregadas por el usuario en el device |
| User-added CA | CA instalada manualmente por el usuario |
| Flavor | Variante de build de la misma aplicación Android |
| `adb reverse` | Puente que redirige un puerto del emulador al host |
| Network Security Config | XML Android que define trust anchors, cleartext y pines |
| APK (Android Package) | Paquete instalable de una aplicación Android |
| `src="system"` | Fuente de CAs del sistema en el NSC |
| `src="user"` | Fuente de CAs agregadas por el usuario en el NSC |
| `<pin-set>` | Lista de pines SPKI aceptados para un dominio |
| SPKI (Subject Public Key Info) | Estructura del certificado que describe la clave pública |
| `debug-overrides` | Configuración especial de confianza para builds debuggable |
| `overridePins` | Define si una anchor puede ignorar pines |
| `SSLHandshakeException` | Excepción que indica un fallo en la negociación TLS |
| `CertPathValidatorException` | Excepción de validación de la cadena hasta una trust anchor |
