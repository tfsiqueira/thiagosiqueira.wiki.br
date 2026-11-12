+++
title = "Cómo identificar y bypassear certificate pinning en Android"
translationKey = "android-certificate-pinning-identification-bypass"
date = "2026-11-11"
author = "Thiago Siqueira"
tags = ["android", "ssl-pinning", "mobile", "frida", "reverse-engineering"]
keywords = ["android", "certificate pinning", "frida", "apktool", "jadx", "okhttp"]
description = "Comparo dos implementaciones de certificate pinning en Android, Network Security Config y el CertificatePinner de OkHttp, para mostrar cómo el análisis estático y la instrumentación dinámica revelan puntos distintos de bypass."
showFullContent = false
readingTime = true
hideComments = false
+++

En el artículo anterior, probé dos cosas importantes:

1. una app Android puede rechazar la CA del Burp antes de que exista cualquier pinning;
2. después de que la cadena del proxy ya es aceptada, un `<pin-set>` todavía puede bloquear la conexión porque el SPKI presentado por el Burp no coincide con el que la app espera.

Esto resuelve una confusión común: no todo fallo de TLS en Android es pinning.

Pero deja una pregunta nueva:

*Si dos apps fallan por pinning, ¿eso significa que la verificación vive en el mismo lugar?*

Esa pregunta importa porque el síntoma visual puede ser casi idéntico: `FAILURE`, con una excepción al establecer la conexión segura.

Aun así, la estrategia de análisis y bypass puede cambiar por completo.

En mi laboratorio, construí dos flavors que rechazan al Burp por pinning:

- `nscPinned`, que usa Network Security Config;
- `okhttpPinned`, que usa el `CertificatePinner` de la biblioteca OkHttp.

Las dos llaman a la misma API, usan los mismos pines y fallan contra el mismo proxy.

Una declara la política en XML, para que la plataforma la aplique. La otra configura la verificación en el código del cliente HTTP.

Esa es la diferencia que quiero investigar en este artículo:

*Cuando la conexión es rechazada, ¿dónde está aplicando la aplicación esa política?*

No introduje OkHttp en el artículo anterior porque ahí la pregunta era otra: quería separar trust store de pinning usando la política nativa de Android. Ahora que esa distinción ya está probada, puedo agregar una segunda implementación sin mezclar los conceptos.

## Antes de empezar: qué sigue igual en el laboratorio

Los comandos de este artículo parten de la raíz del proyecto, con los APKs ya compilados e instalados, la API local en ejecución y las CAs del laboratorio y del Burp instaladas en el user store del Android.

Uso un emulador Android API 36 con root, OkHttp **4.12.0**, Frida y `frida-server` **17.17.0**, frida-tools **14.10.4**, apktool **3.0.3** y JADX **1.5.5**. El `frida-server` ya está en ejecución con privilegios de root; es el componente en el device que le permite al cliente Frida instrumentar el proceso. Tener el proxy funcionando, por sí solo, no da ese acceso.

La preparación está descrita en [Entorno del laboratorio](#TODO-link-github-docs-00-environment). Antes de los comandos, cargo las rutas del SDK de Android y de Java:

```bash
source scripts/lab-env.sh
```

Mantengo el mismo entorno del artículo anterior: **Android Emulator → Burp Suite en macOS → API HTTPS local**.

La API sigue expuesta en `https://api.ssl-pinning.127.0.0.1.nip.io:8443/profile`.

El proxy sigue configurado en el Android:

```bash
adb shell settings put global http_proxy 10.0.2.2:8080
```

Y el puente del emulador hasta la API local sigue activo:

```bash
adb reverse tcp:8443 tcp:8443
```

También sigo usando la misma app con múltiples flavors. Comparten la pantalla y el endpoint. Las dos variantes con pinning comparadas aquí, `nscPinned` y `okhttpPinned`, aplican los mismos pines:

- `sha256/Kyfpm+U4uD7FC1lNrw/7LHwerRcQ0EIMmaWCZd0dk8Q=`
- `sha256/jN72Sgrbs/bv6VCez+cHlzrPx3iW26rCyaTwmAJB0Tk=`

Esto es importante porque me permite comparar solo el punto de implementación.

La infraestructura se mantiene igual; lo que cambia es el punto de implementación de la política.

## Antes de comparar: la baseline todavía pasa

Antes de comparar dos fallos, revalido rápidamente la baseline del artículo anterior.

La flavor `proxyTrusted` acepta la CA del Burp y no tiene pinning:

```bash
adb shell am force-stop com.example.sslpinninglab.proxytrusted
adb shell am start \
  -n com.example.sslpinninglab.proxytrusted/com.example.sslpinninglab.MainActivity \
  --ez auto_request true
```

El resultado sigue siendo `SUCCESS`, con `HTTP 200`.

Esto importa porque mantiene la misma conclusión del artículo anterior: *si otra flavor falla ahora, la causa ya no es simplemente "el Burp no funciona".*

## La primera evidencia: dos flavors fallan contra el mismo Burp

Empiezo repitiendo el fallo de `nscPinned`, que ya apareció en el artículo anterior:

```bash
adb shell am force-stop com.example.sslpinninglab.nscpinned
adb shell am start \
  -n com.example.sslpinninglab.nscpinned/com.example.sslpinninglab.MainActivity \
  --ez auto_request true
```

El resultado es `FAILURE`, con `Exception: SSLHandshakeException` y causa raíz `CertificateException: Pin verification failed`.

Para seguir las excepciones emitidas por la app, abro otra terminal antes de la solicitud:

```bash
adb logcat -v threadtime -s 'SslPinningLab:E' '*:S'
```

`SslPinningLab` es la etiqueta que usa el código del laboratorio para registrar la excepción completa. El filtro selecciona sus errores y silencia las demás etiquetas. Puede incluir registros anteriores; para correlacionar el fallo, observo el horario de la nueva solicitud. Cierro la lectura con `Ctrl+C`.

El fragmento relevante muestra `Caused by: java.security.cert.CertificateException: Pin verification failed`, en `NetworkSecurityTrustManager.checkPins(...)`.

Después abro `okhttpPinned`:

```bash
adb shell am force-stop com.example.sslpinninglab.okhttppinned
adb shell am start \
  -n com.example.sslpinninglab.okhttppinned/com.example.sslpinninglab.MainActivity \
  --ez auto_request true
```

En la pantalla, la app también falla, con `FAILURE`, `Exception: SSLPeerUnverifiedException` y causa raíz `SSLPeerUnverifiedException: Certificate pinning failure!`.

El logcat confirma `javax.net.ssl.SSLPeerUnverifiedException: Certificate pinning failure!`.

A primera vista, las dos variantes parecen contar la misma historia:

- el Burp presenta una clave pública distinta;
- la app esperaba los SPKIs legítimos;
- la solicitud HTTP de la aplicación no llega a enviarse por el túnel TLS.

Esa distinción de capas importa: el proxy recibe un `CONNECT` HTTP para abrir el túnel incluso antes de la validación TLS. Lo que queda bloqueado aquí es el `GET /profile` de la aplicación.

En OkHttp 4.12.0, el `CertificatePinner` se llama después del handshake del socket y de la verificación de hostname, antes de liberar la conexión para la solicitud de la aplicación. El NSC, en cambio, aplica los pines durante la validación de confianza de la plataforma. Entonces, fallar al establecer una conexión segura no significa que los dos rechazos ocurran en la misma etapa.

Las excepciones dan una pista de que la implementación cambió:

| Variante | Excepción principal | Punto visible en el stack trace |
| --- | --- | --- |
| `nscPinned` | `SSLHandshakeException`, causada por `CertificateException` | `NetworkSecurityTrustManager.checkPins(...)` |
| `okhttpPinned` | `SSLPeerUnverifiedException` | `CertificatePinner.check$okhttp(...)` |

El problema lógico parece el mismo.

El punto de decisión todavía necesita confirmarse.

Esta es la primera idea importante del artículo:

*El síntoma puede ser igual, pero la implementación cambia la investigación.*

<img src="/images/posts/03-como-identificar-e-bypassar-certificate-pinning-em-android/pinning-failures-comparison.png" alt="Comparación entre las flavors nscPinned y okhttpPinned fallando contra el Burp por mecanismos distintos" class="center" width="820">

*Las dos flavors fallan contra el mismo Burp, pero el mensaje ya sugiere que la decisión se toma en puntos distintos.*

En este punto, todavía no trato la excepción como prueba final.

Un fallo de handshake es una pista. Me ayuda a decidir dónde buscar, pero no reemplaza la inspección del APK.

| Evidencia | Qué sugiere | Qué todavía necesito confirmar |
| --- | --- | --- |
| Fallo al pasar por el Burp | La conexión se rechazó antes del HTTP normal de la app | Qué validación tomó la decisión |
| `NetworkSecurityTrustManager.checkPins(...)` en el stack trace | La política puede venir del Network Security Config | Si el XML realmente contiene un `<pin-set>` |
| `CertificatePinner.check$okhttp(...)` en el stack trace | La biblioteca OkHttp puede estar aplicando pines | Si esa llamada participa del flujo real del APK |

## Pregunta 1: ¿dónde busco primero dentro del APK?

Cuando tengo un APK y quiero entender dónde vive el pinning, no empiezo tratando de correr un script universal de bypass.

Empiezo buscando evidencia estática.

Para eso, uso dos herramientas con roles distintos:

| Herramienta | Qué quiero de ella |
| --- | --- |
| `apktool` | Manifest, resources XML y smali decodificados del APK |
| `jadx` | Una vista Java/Kotlin aproximada del bytecode |

Organizo los archivos generados por experimento, dentro de `/tmp/ssl-pinning-lab`:

| Directorio | Contenido |
| --- | --- |
| `/tmp/ssl-pinning-lab/static-analysis` | Recursos decodificados por apktool y código exportado por JADX |
| `/tmp/ssl-pinning-lab/nsc-patch` | Copia editable del APK NSC y APKs reconstruidos |

Estos directorios son áreas de trabajo locales, separadas de las fuentes en `lab/`. Los nombres identifican el experimento y pueden usarse tanto en este artículo como en la reproducción del laboratorio. Como quedan en `/tmp`, el sistema puede eliminarlos; no guardo ahí la única copia de una evidencia que quiera preservar.

Decodifico los tres APKs principales del laboratorio:

```bash
mkdir -p /tmp/ssl-pinning-lab/static-analysis/framework

apktool d -f \
  -p /tmp/ssl-pinning-lab/static-analysis/framework \
  lab/android-app/app/build/outputs/apk/proxyTrusted/debug/app-proxyTrusted-debug.apk \
  -o /tmp/ssl-pinning-lab/static-analysis/apktool/proxyTrusted

apktool d -f \
  -p /tmp/ssl-pinning-lab/static-analysis/framework \
  lab/android-app/app/build/outputs/apk/nscPinned/debug/app-nscPinned-debug.apk \
  -o /tmp/ssl-pinning-lab/static-analysis/apktool/nscPinned

apktool d -f \
  -p /tmp/ssl-pinning-lab/static-analysis/framework \
  lab/android-app/app/build/outputs/apk/okhttpPinned/debug/app-okhttpPinned-debug.apk \
  -o /tmp/ssl-pinning-lab/static-analysis/apktool/okhttpPinned
```

El comando `apktool d` decodifica el APK.

No recupera el proyecto original.

Lo que me entrega es una representación reconstruida de los recursos y del bytecode, suficiente para seguir referencias como el manifest, el XML y el smali.

Los parámetros más importantes son:

| Fragmento | Función |
| --- | --- |
| `-f` | Sobrescribe una extracción anterior |
| `-p /tmp/.../framework` | Mantiene los frameworks de apktool en una carpeta aislada |
| `-o /tmp/...` | Define el directorio donde se graban los recursos decodificados |

Después abro la variante OkHttp en jadx:

```bash
jadx -d /tmp/ssl-pinning-lab/static-analysis/jadx/okhttpPinned \
  lab/android-app/app/build/outputs/apk/okhttpPinned/debug/app-okhttpPinned-debug.apk
```

Para localizar la referencia al NSC en los manifests decodificados, uso:

```bash
rg -n 'networkSecurityConfig' /tmp/ssl-pinning-lab/static-analysis/apktool/*/AndroidManifest.xml
```

La búsqueda apunta al siguiente atributo en el `AndroidManifest.xml` de las tres variantes:

```xml
<application
    android:networkSecurityConfig="@xml/network_security_config"
    ... />
```

Esto todavía no prueba pinning.

Solo prueba que existe un Network Security Config activo que vale la pena leer.

Encontrar un archivo llamado `network_security_config.xml` por sí solo no bastaría. Lo que importa es confirmar que el manifest realmente apunta a él.

## Pregunta 2: ¿qué revela el XML sobre `nscPinned`?

Sigo la referencia del manifest hasta el recurso decodificado:

```bash
cat /tmp/ssl-pinning-lab/static-analysis/apktool/proxyTrusted/res/xml/network_security_config.xml
cat /tmp/ssl-pinning-lab/static-analysis/apktool/nscPinned/res/xml/network_security_config.xml
```

En la baseline `proxyTrusted`, el XML extraído por apktool muestra:

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

No hay `<pin-set>`.

Esta variante acepta la CA del Burp, pero no exige un SPKI específico.

En `nscPinned`, el XML es casi igual, pero gana un `domain-config` con pines:

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

En este caso, el APK me cuenta casi todo sin que tenga que abrir ninguna clase Java:

1. la app acepta CAs del sistema y del user store;
2. el dominio de la API está explícitamente cubierto por pines;
3. los hashes aparecen directamente en el XML;
4. el punto de bloqueo es una política declarativa de Android.

Esto también explica por qué el stack trace apuntaba a `NetworkSecurityTrustManager.checkPins(...)`.

Quien aplica la decisión no es una clase escrita por la app.

Es la propia plataforma Android, leyendo la política del XML.

En este punto, el bypass más natural ya empieza a aparecer: *si el pin vive en el XML, quitar el pin del XML es una hipótesis razonable.*

## Pregunta 3: ¿y si el XML no tiene `<pin-set>`?

Ahora abro el XML de `okhttpPinned`:

```bash
cat /tmp/ssl-pinning-lab/static-analysis/apktool/okhttpPinned/res/xml/network_security_config.xml
```

Es igual al de la baseline:

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

No hay `<pin-set>`.

Si parara aquí, concluiría mal: *esta app no tiene pinning.*

Pero ya vi a la app fallar con `SSLPeerUnverifiedException: Certificate pinning failure!`.

Esto me indica que la decisión vive en otro lugar.

En los archivos exportados por JADX, busco tanto la construcción de los pines como su asociación con el cliente:

```bash
rg -n 'CertificatePinner|certificatePinner' /tmp/ssl-pinning-lab/static-analysis/jadx/okhttpPinned/sources
```

La búsqueda apunta a `MainActivity`. Encontrar la clase de la biblioteca empaquetada en el APK no bastaría: necesito seguir su uso en el cliente que ejecuta la solicitud.

Para inspeccionar ese fragmento en la interfaz gráfica, también puedo abrir el APK directamente:

```bash
jadx-gui lab/android-app/app/build/outputs/apk/okhttpPinned/debug/app-okhttpPinned-debug.apk
```

En el árbol de código, abro `com.example.sslpinninglab.MainActivity` y localizo `requestWithOkHttpPinning`. El fragmento relevante, formateado para lectura, es:

```java
CertificatePinner certificatePinner = new CertificatePinner.Builder()
        .add(BuildConfig.API_HOST, BuildConfig.LEAF_PIN, BuildConfig.BACKUP_PIN)
        .build();

OkHttpClient client = new OkHttpClient.Builder()
        .certificatePinner(certificatePinner)
        .build();
```

Ahora la diferencia queda clara:

| Caso | Dónde vive el pin | Quién aplica la validación |
| --- | --- | --- |
| `nscPinned` | `res/xml/network_security_config.xml` | Android / Network Security Config |
| `okhttpPinned` | Código Java/Kotlin | `okhttp3.CertificatePinner` |

El `<pin-set>` del Network Security Config declara los pines en XML; el NSC también configura otras políticas de red y puede existir sin pinning. El `CertificatePinner`, en cambio, configura los pines programáticamente.

En los dos casos, la app está pidiendo lo mismo: *aceptar solo una cadena que contenga un SPKI esperado.*

Pero la forma en que esa regla entra al APK es distinta.

Y esa diferencia cambia el bypass.

<img src="/images/posts/03-como-identificar-e-bypassar-certificate-pinning-em-android/jadx-okhttp-certificate-pinner.png" alt="JADX mostrando el CertificatePinner construyéndose dentro de MainActivity" class="center" width="820">

*En JADX, la política de pinning aparece en el propio código de la app: `CertificatePinner.Builder().add(...)` arma los pines y el cliente OkHttp recibe esa configuración.*

## Pregunta 4: ¿por qué no empiezo por un script universal?

Después de encontrar `CertificatePinner`, sería tentador descargar un script listo y esperar que funcione.

A veces funciona.

Pero eso no enseña dónde ocurre la decisión.

En mi laboratorio, quiero probar el menor cambio necesario. Entonces parto del stack trace.

El fallo de la flavor OkHttp muestra `SSLPeerUnverifiedException: Certificate pinning failure!`, en `okhttp3.CertificatePinner.check$okhttp(...)`.

Ese detalle importa, pero no porque `check$okhttp` sea una API universal.

Es el método que aparece en esta versión de OkHttp y en este camino de ejecución del laboratorio. En otra versión, en otro build o en una app ofuscada, el nombre o la firma pueden cambiar.

`CertificatePinner` tiene más de un método relacionado con esta verificación:

- `check(java.lang.String, java.util.List)`
- `check(java.lang.String, java.security.cert.Certificate[])`
- `check$okhttp(java.lang.String, kotlin.jvm.functions.Function0)`

El stack trace ya indica qué método participó en el fallo. Ahora quiero confirmar si está disponible en el proceso con la firma que pretendo interceptar. Enumerar métodos confirma su existencia; no demuestra, por sí solo, que se ejecutaron.

Por eso escribo un script de inspección:

```javascript
Java.perform(function () {
  const CertificatePinner = Java.use("okhttp3.CertificatePinner");
  const methods = CertificatePinner.class.getDeclaredMethods();

  console.log("[*] okhttp3.CertificatePinner declared methods:");
  for (let i = 0; i < methods.length; i += 1) {
    console.log("    " + methods[i].toString());
  }
});
```

Este script solo inspecciona los métodos. Responde a la pregunta: *¿qué métodos expone realmente esta clase en el proceso que está corriendo?*

Lo ejecuto:

```bash
frida -U -n 'SSL Lab - OkHttp Pinned' -l lab/frida/inspect-okhttp.js -q
```

Los parámetros significan:

| Fragmento | Función |
| --- | --- |
| `-U` | Conecta al device USB, en este caso el emulador |
| `-n 'SSL Lab - OkHttp Pinned'` | Se adjunta al proceso ya abierto por nombre |
| `-l lab/frida/inspect-okhttp.js` | Carga el script JavaScript |
| `-q` | Ejecuta en modo quiet, dejando la salida más limpia |

La salida muestra `[*] okhttp3.CertificatePinner declared methods:`, seguido de:

- `public final void okhttp3.CertificatePinner.check(java.lang.String,java.util.List)`
- `public final void okhttp3.CertificatePinner.check(java.lang.String,java.security.cert.Certificate[])`
- `public final void okhttp3.CertificatePinner.check$okhttp(java.lang.String,kotlin.jvm.functions.Function0)`

Ahora tengo dos evidencias que apuntan al mismo lugar:

1. el stack trace del fallo pasa por `check$okhttp`;
2. el runtime confirma que ese método existe con la firma esperada.

Con el fallo localizado y la firma confirmada, puedo preparar el hook dirigido.

El script también queda en el laboratorio en `lab/frida/inspect-okhttp.js`.

Así, la salida de la captura de pantalla no se vuelve la única evidencia reproducible de la investigación.

## Pregunta 5: ¿cuál es el hook más pequeño que prueba la hipótesis?

Antes del hook, una precondición importa: en esta flavor, la CA del Burp ya es aceptada por la política de confianza de la app.

Si eso no fuera cierto, quitar solo el pinning no bastaría.

El hook que voy a probar no convierte una cadena inválida en válida. Solo impide la verificación adicional de SPKI hecha por el `CertificatePinner`.

Si `check$okhttp` es el punto donde la biblioteca rechaza el SPKI presentado, no necesito reescribir toda la pila TLS.

Solo necesito neutralizar esa decisión.

El hook queda así:

```javascript
Java.perform(function () {
  const CertificatePinner = Java.use("okhttp3.CertificatePinner");

  const checkOkHttp = CertificatePinner["check$okhttp"].overload(
    "java.lang.String",
    "kotlin.jvm.functions.Function0",
  );

  checkOkHttp.implementation = function (hostname, peerCertificatesFn) {
    console.log("[+] Bypassing CertificatePinner.check$okhttp for " + hostname);
    return;
  };

  console.log("[*] Targeted OkHttp CertificatePinner hook installed");
});
```

Para entender por qué este hook funciona, necesito mirar el contrato del método original. `check$okhttp` no devuelve un booleano que diga si los pines fueron aceptados. Retorna normalmente cuando la verificación permite continuar y lanza `SSLPeerUnverifiedException` cuando rechaza los pines.

El segundo parámetro, `Function0`, es una función de Kotlin sin argumentos que entrega la lista de certificados para la verificación. Al reemplazar el método entero, mi hook no llama a esa función ni ejecuta la comparación original.

El `return;` termina un método cuyo retorno en Java es `void`. Para quien llama, la invocación termina normalmente, sin la excepción que interrumpiría la conexión. Eso es lo que neutraliza la restricción en este flujo; no hay una nueva validación de los certificados dentro del hook.

El script hace cuatro cosas:

1. espera al runtime de Java y ejecuta la función de instrumentación con `Java.perform`;
2. obtiene la clase `okhttp3.CertificatePinner`;
3. selecciona exactamente el overload observado en el runtime;
4. reemplaza la implementación por un retorno vacío.

Adjunto el hook:

```bash
frida -U -n 'SSL Lab - OkHttp Pinned' -l lab/frida/targeted-okhttp-hook.js
```

Este segundo script también está versionado en el laboratorio en `lab/frida/targeted-okhttp-hook.js`.

Mantengo esa sesión abierta y espero la confirmación de que el hook fue instalado: `[*] Targeted OkHttp CertificatePinner hook installed`.

Solo después toco de nuevo **REQUEST PROFILE**, en la pantalla de la app ya instrumentada. Eso dispara una nueva llamada sin terminar el proceso al que el Frida está adjunto. No uso `force-stop` en este momento, porque eso también terminaría el proceso instrumentado.

El Frida muestra de nuevo `[*] Targeted OkHttp CertificatePinner hook installed` y, a continuación, `[+] Bypassing CertificatePinner.check$okhttp for api.ssl-pinning.127.0.0.1.nip.io`.

Y la app cambia de estado a `SUCCESS`, con `HTTP 200`.

El payload vuelve a aparecer en la pantalla.

<img src="/images/posts/03-como-identificar-e-bypassar-certificate-pinning-em-android/okhttp-after-hook.png" alt="Flavor okhttpPinned funcionando vía Burp después del hook en CertificatePinner" class="center" width="420">

*Después del hook, la misma flavor vuelve a responder con `HTTP 200` y el payload de la API.*

También reviso el historial HTTP del Burp. La entrada muestra `GET /profile`, `User-Agent: okhttp/4.12.0` y la respuesta `HTTP/1.1 200 OK`. El campo `time` del JSON, `2026-11-11T22:57:50.385Z`, coincide con el payload que muestra la app después del hook. Esa correlación une la ejecución instrumentada con la solicitud y la respuesta legibles en el proxy.

<img src="/images/posts/03-como-identificar-e-bypassar-certificate-pinning-em-android/burp-okhttp-after-hook.png" alt="Historial del Burp mostrando GET /profile con User-Agent okhttp/4.12.0 y respuesta HTTP 200 cuyo horario corresponde al éxito después del hook" class="center" width="1000">

*La entrada 2856 del Burp registra la llamada OkHttp después del hook. El horario en el JSON es el mismo observado en la app.*

Quité la verificación adicional de pinning hecha por la biblioteca. El handshake sigue ocurriendo y la CA del Burp sigue siendo aceptada por la política de confianza de la app; lo que cambió fue la decisión local que comparaba los SPKIs.

Esto confirma el papel del `CertificatePinner` en este flujo específico. Fallos en otras verificaciones TLS siguen exigiendo su propia investigación.

### ¿Qué pasa cuando reinicio sin el hook?

Para comprobar si el cambio quedó restringido al proceso, cierro la sesión del Frida con `Ctrl+C` y detengo la app. Después abro de nuevo el mismo APK, sin adjuntar el script:

```bash
adb shell am force-stop com.example.sslpinninglab.okhttppinned
adb shell am start \
  -n com.example.sslpinninglab.okhttppinned/com.example.sslpinninglab.MainActivity \
  --ez auto_request true
```

El resultado vuelve a ser `FAILURE`, con `SSLPeerUnverifiedException: Certificate pinning failure!`.

El proxy y el APK siguen siendo los mismos. Al terminar el proceso instrumentado, la implementación original vuelve a usarse en la próxima ejecución. La secuencia observada queda:

| Estado | Resultado |
| --- | --- |
| `proxyTrusted`, sin pinning, vía Burp | `SUCCESS`, HTTP 200 |
| `okhttpPinned`, antes del hook | `FAILURE`, `Certificate pinning failure!` |
| `okhttpPinned`, con el hook ejecutado | `SUCCESS`, HTTP 200 |
| El mismo APK reiniciado sin el hook | `FAILURE`, `Certificate pinning failure!` |

<img src="/images/posts/03-como-identificar-e-bypassar-certificate-pinning-em-android/okhttp-restarted-without-hook.png" alt="Flavor okhttpPinned volviendo a rechazar los pines del Burp después del reinicio sin instrumentación" class="center" width="420">

*El reinicio sin hook restaura el rechazo. La instrumentación anterior no alteró el APK instalado.*

Las pantallas, los XMLs y la línea de tiempo de esta validación están en las [evidencias complementarias](#TODO-link-github-evidence-article-03-review-readme). El log del Frida registra la ejecución del hook para el hostname de la API; la respuesta que muestra la app contiene el horario de la nueva solicitud. La captura del historial del Burp permite correlacionar ese mismo payload con el HTTP observado en el proxy.

## Pregunta 6: ¿el bypass de `nscPinned` sigue la misma lógica?

La lógica sigue siendo neutralizar la restricción adicional, pero elijo otro punto de intervención.

En `nscPinned`, la política se declara en el XML y la ejecuta código de la plataforma Android, como muestra `NetworkSecurityTrustManager.checkPins`. Para este experimento, el bypass más didáctico es quitar el `<pin-set>` antes de la ejecución.

Esto no significa que el NSC solo pueda contornarse con un patch, ni que OkHttp solo pueda contornarse con un hook. También sería posible investigar la validación de la plataforma en runtime, o alterar en el APK el código que configura OkHttp. Aquí elijo dos intervenciones para comparar un cambio en memoria con un cambio en el artefacto.

Altero el artefacto, reconstruyo el APK y solo después vuelvo a ejecutar la app.

Preparo un área separada del análisis estático y decodifico el APK:

```bash
mkdir -p /tmp/ssl-pinning-lab/nsc-patch/framework

apktool d -f \
  -p /tmp/ssl-pinning-lab/nsc-patch/framework \
  lab/android-app/app/build/outputs/apk/nscPinned/debug/app-nscPinned-debug.apk \
  -o /tmp/ssl-pinning-lab/nsc-patch/nscPinned-decoded
```

El archivo relevante aparece en `/tmp/ssl-pinning-lab/nsc-patch/nscPinned-decoded/res/xml/network_security_config.xml`.

El fragmento original contiene:

```xml
<pin-set expiration="2035-01-01">
    <pin digest="SHA-256">Kyfpm+U4uD7FC1lNrw/7LHwerRcQ0EIMmaWCZd0dk8Q=</pin>
    <pin digest="SHA-256">jN72Sgrbs/bv6VCez+cHlzrPx3iW26rCyaTwmAJB0Tk=</pin>
</pin-set>
```

Quito solo ese bloque y mantengo el resto de la política igual. Para verificar el alcance de la edición, comparo el XML original del proyecto con el recurso que acabo de modificar:

```bash
diff -u \
  lab/android-app/app/src/nscPinned/res/xml/network_security_config.xml \
  /tmp/ssl-pinning-lab/nsc-patch/nscPinned-decoded/res/xml/network_security_config.xml
```

La comparación puede incluir diferencias de formato de apktool; el cambio de política que quiero ver es la eliminación del `<pin-set>`, preservando los dominios y las trust anchors.

En el APK de laboratorio que estoy reconstruyendo, esto prueba el efecto de quitar la política del artefacto modificado. No cambia el APK original que existía antes de la edición.

Después reconstruyo el APK:

```bash
apktool b -p /tmp/ssl-pinning-lab/nsc-patch/framework \
  /tmp/ssl-pinning-lab/nsc-patch/nscPinned-decoded \
  -o /tmp/ssl-pinning-lab/nsc-patch/app-nscPinned-patched-unsigned.apk
```

El APK reconstruido todavía necesita alinearse y firmarse. Antes de ejecutar los comandos, separo el papel de cada etapa:

| Comando | Qué hace en este experimento |
| --- | --- |
| `zipalign` | Ajusta la alineación de los datos no comprimidos dentro del ZIP que forma el APK, permitiendo un acceso eficiente a esos archivos. No firma el paquete. |
| `apksigner sign` | Firma el APK reconstruido con la clave privada de la debug keystore e incluye el certificado del firmante. La firma le permite al Android verificar la integridad del paquete e identificar la clave que lo firmó. |
| `apksigner verify --print-certs` | Verifica la firma del APK y muestra información sobre el certificado de firma. No prueba el funcionamiento de la app ni su pinning TLS. |

Ejecuto las etapas en este orden: alineación, firma y verificación.

```bash
zipalign -f -p 4 \
  /tmp/ssl-pinning-lab/nsc-patch/app-nscPinned-patched-unsigned.apk \
  /tmp/ssl-pinning-lab/nsc-patch/app-nscPinned-patched-aligned.apk

apksigner sign \
  --ks ~/.android/debug.keystore \
  --ks-key-alias androiddebugkey \
  --ks-pass pass:android \
  --key-pass pass:android \
  --out /tmp/ssl-pinning-lab/nsc-patch/app-nscPinned-patched.apk \
  /tmp/ssl-pinning-lab/nsc-patch/app-nscPinned-patched-aligned.apk

apksigner verify --print-certs \
  /tmp/ssl-pinning-lab/nsc-patch/app-nscPinned-patched.apk
```

Alterar el recurso protegido por la firma invalida la firma anterior; por eso el APK reconstruido necesita firmarse de nuevo. Alineo antes de firmar, porque la alineación modifica el paquete y, si se hiciera después, invalidaría la firma producida por `apksigner`. La verificación final comprueba el resultado antes de la instalación. Este orden está descrito en las referencias oficiales de [zipalign](https://developer.android.com/tools/zipalign) y [apksigner](https://developer.android.com/tools/apksigner).

El certificado de firma del APK tiene una función distinta a la del certificado TLS de la API: el primero identifica la clave usada para firmar el paquete; el segundo asocia la clave pública del servidor al dominio en la autenticación TLS. Firmar el APK alterado no cambia el certificado de la API ni los pines que la app espera encontrar.

En el laboratorio, el build original y el APK patched usan la misma debug keystore, `~/.android/debug.keystore`.

Por eso `adb install -r` logra reemplazar la app instalada sin exigir desinstalación.

En una app real, un APK modificado normalmente no podría reemplazar la instalación oficial, porque el atacante no tendría la clave privada de firma de producción. El bypass sigue siendo relevante cuando la precondición incluye control sobre la instalación, el device o el entorno de prueba.

Después instalo el APK alterado:

```bash
adb install -r /tmp/ssl-pinning-lab/nsc-patch/app-nscPinned-patched.apk
adb shell am force-stop com.example.sslpinninglab.nscpinned
adb shell am start \
  -n com.example.sslpinninglab.nscpinned/com.example.sslpinninglab.MainActivity \
  --ez auto_request true
```

El resultado cambia a `SUCCESS`, con `HTTP 200`.

De nuevo, no hice que el Burp fuera "más confiable".

Solo quité la decisión adicional que rechazaba el SPKI presentado por el proxy.

<img src="/images/posts/03-como-identificar-e-bypassar-certificate-pinning-em-android/nsc-patch-before-after.png" alt="Comparación entre nscPinned antes y después de la eliminación del pin-set en el APK" class="center" width="820">

*Antes del patch, la política declarativa todavía rechaza el SPKI del Burp. Después de quitar el `<pin-set>` y reinstalar el APK, la misma solicitud pasa.*

## Lo que revela la comparación entre los dos bypasses

La diferencia central del laboratorio queda así:

| Variante | Dónde vive el pinning | Evidencia estática | Evidencia dinámica | Bypass demostrado |
| --- | --- | --- | --- | --- |
| `nscPinned` | XML | `<pin-set>` en `network_security_config.xml` | `NetworkSecurityTrustManager.checkPins` | Quitar el `<pin-set>` y reconstruir el APK |
| `okhttpPinned` | Código Java/Kotlin | `CertificatePinner.Builder().add(...)` | `CertificatePinner.check$okhttp` | Hook de Frida en el overload real |

Esa tabla responde la pregunta principal del artículo: *dos apps pueden fallar por el mismo motivo lógico y ofrecer puntos distintos de intervención.*

Las técnicas usadas aquí dependen de la implementación y del acceso disponible. Cambios de versión, ofuscación, clases personalizadas o validaciones nativas pueden exigir otros puntos de intervención. La asociación entre XML y patch, o entre OkHttp y hook, describe estos experimentos; no es una regla exclusiva para analizar cualquier APK.

Lo que sigue siendo válido en todos estos casos es la metodología:

1. identificar la implementación;
2. localizar el punto exacto de decisión;
3. elegir el menor bypass que pruebe la hipótesis;
4. validar si el comportamiento cambió.

## Fuentes para revisar los mecanismos

- [Android — Network Security Configuration](https://developer.android.com/privacy-and-security/security-config): políticas de confianza y pinning declaradas en el XML.
- [OkHttp 4.12.0 — CertificatePinner](https://github.com/square/okhttp/blob/parent-4.12.0/okhttp/src/main/kotlin/okhttp3/CertificatePinner.kt): comparación de los pines, retorno normal y excepción de rechazo.
- [OkHttp 4.12.0 — RealConnection](https://github.com/square/okhttp/blob/parent-4.12.0/okhttp/src/main/kotlin/okhttp3/internal/connection/RealConnection.kt): orden entre handshake, verificación de hostname y pinning.
- [Frida — JavaScript API](https://frida.re/docs/javascript-api/): uso de `Java.perform`, `Java.use` y sustitución de métodos.

## Lo que prueban estos bypasses

Al final del experimento, puedo probar:

1. `nscPinned` y `okhttpPinned` bloquean el mismo MITM por mecanismos distintos;
2. el análisis estático es lo que revela dónde vive la regla de pinning;
3. el stack trace ayuda a formular la hipótesis sobre el punto dinámico real del fallo;
4. un hook de Frida dirigido puede neutralizar la decisión de una biblioteca en runtime;
5. un patch de APK puede quitar una política declarativa antes de la ejecución;
6. el pinning aumenta el costo del MITM, pero no es una barrera absoluta contra alguien que controla el proceso o el APK.

## Modelo mental final

Después de esta etapa del laboratorio, la forma más útil de pensar en bypass es:

- *El error visual me dice que hay un fallo.*
- *El stack trace me ayuda a localizar dónde buscar.*
- *El APK confirma cómo se implementó.*
- *El bypass necesita atacar exactamente ese punto.*

O, de forma todavía más corta:

- *La implementación revela dónde intervenir.*
- *El acceso disponible orienta cómo intervenir.*
- *El cambio observado prueba la hipótesis.*

El valor no está en memorizar un script.

El valor está en observar el fallo, localizar la política, confirmar cómo participa en el flujo y solo entonces elegir la técnica de prueba.

Después de probar que el bypass depende de la implementación, la próxima pregunta deja de ser "¿cómo quito el pinning?" y pasa a ser: *¿qué pasa cuando la clave legítima del servidor necesita cambiar?*

Esa es la investigación que quiero abrir en el próximo artículo.

## Referencia rápida de los términos usados

| Término | Significado |
| --- | --- |
| APK (Android Package) | Paquete instalable de una aplicación Android |
| apktool | Herramienta para decodificar y reconstruir APKs |
| jadx | Decompilador que reconstruye una vista Java/Kotlin del bytecode |
| Network Security Config | XML Android que define trust anchors, cleartext y pines |
| `<pin-set>` | Lista de pines SPKI aceptados para un dominio |
| `CertificatePinner` | Clase de OkHttp usada para aplicar pinning programático |
| Frida | Toolkit de instrumentación dinámica |
| Hook | Sustitución de un comportamiento en runtime |
| Overload | Variante de un método con firma distinta |
| `check$okhttp` | Método interno de OkHttp observado en el runtime del laboratorio |
| `SSLPeerUnverifiedException` | Excepción de verificación de la identidad del peer; en este flujo, el mensaje y el stack trace apuntan al rechazo de los pines |
| `NetworkSecurityTrustManager.checkPins` | Método de la plataforma Android que aplica los pines del NSC |
| `zipalign` | Herramienta que alinea el APK antes de la firma |
| `apksigner` | Herramienta que firma y valida APKs |
| Debug keystore | Clave de firma usada en builds Android de debug |
