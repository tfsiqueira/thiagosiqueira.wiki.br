+++
title = "Cómo un cliente valida la autenticidad de un dominio durante el handshake TLS"
translationKey = "tls-handshake-domain-validation"
date = "2026-10-10"
author = "Thiago Siqueira"
tags = ["tls", "pki", "criptografia", "openssl", "redes"]
keywords = ["tls", "pki", "certificados", "openssl", "handshake"]
description = "Una PoC real con mercadopago.com.br para entender, paso a paso, cómo OpenSSL valida la cadena de certificados de un dominio y qué significa realmente confiar en una trust anchor local."
showFullContent = false
readingTime = true
hideComments = false
+++

Cuando abrimos `https://mercadopago.com.br` en el navegador, o cuando una aplicación hace una solicitud HTTPS (Hypertext Transfer Protocol Secure) a una API (Application Programming Interface), existe una pregunta que necesita responderse antes de intercambiar cualquier dato sensible.

Antes de que la conexión sea aceptada, cliente y servidor pasan por el handshake, la negociación inicial del TLS en la que se acuerdan parámetros criptográficos y se valida la identidad del servidor. Es en ese momento que surge la pregunta:

*¿Cómo sabe el cliente que está hablando con el servidor legítimo de ese dominio?*

Es común resumir TLS (Transport Layer Security) como "el protocolo que cifra la conexión". Esa definición no está mal, pero está incompleta. El cifrado, por sí solo, no prueba identidad. Un atacante también puede establecer una conexión cifrada con el cliente. Lo que importa es saber si la clave pública presentada del otro lado pertenece, de hecho, al dominio al que el cliente pretendía acceder.

Este mecanismo depende de una infraestructura de claves públicas, o PKI (Public Key Infrastructure). Dentro de ella, las autoridades certificadoras, o CAs (Certificate Authorities), firman certificados para vincular nombres de dominio con claves públicas.

Este artículo parte de una PoC (Proof of Concept) real que realicé con `mercadopago.com.br`. La idea no es memorizar la cadena específica observada, porque los certificados públicos cambian con el tiempo. La idea es seguir la investigación paso a paso y responder, en cada etapa, qué es lo que el cliente está realmente verificando.

El hilo conductor será siempre el mismo:

- *¿Qué envió el servidor?*
- *¿Qué verificó el cliente?*
- *¿En qué momento decidió confiar?*

## Antes de empezar: qué quise reproducir

Mi objetivo es capturar la cadena presentada por un servidor real y entender cómo OpenSSL logra validarla localmente.

El experimento usa:

- `openssl s_client` para abrir una conexión TLS;
- `openssl x509` para inspeccionar certificados;
- `openssl verify` para reproducir la validación;
- el trust store local de OpenSSL instalado vía Homebrew, es decir, el repositorio local de CAs que ese OpenSSL ya acepta como confiables.

Creo un directorio de trabajo:

```bash
mkdir -p ~/tmp/cert-lab-mercadopago
cd ~/tmp/cert-lab-mercadopago

openssl version -a
```

En la máquina usada para el ejercicio, la salida relevante fue `OPENSSLDIR: "/opt/homebrew/etc/openssl@3"`.

Ese detalle parece pequeño, pero será importante más adelante. El OpenSSL de Homebrew usa su propio bundle de CAs. No necesariamente consulta el mismo trust store usado por las APIs nativas de macOS, por Chrome, por Java o por una aplicación Android.

## Pregunta 1: ¿qué envía realmente el servidor?

Empiezo abriendo una conexión TLS con el servidor y pidiéndole a OpenSSL que muestre todos los certificados presentados:

```bash
openssl s_client \
  -connect mercadopago.com.br:443 \
  -servername mercadopago.com.br \
  -showcerts \
  </dev/null 2>&1 | tee s_client.txt
```

Cada parte del comando cumple un rol:

| Fragmento | Función |
|---|---|
| `-connect mercadopago.com.br:443` | Define host y puerto |
| `-servername mercadopago.com.br` | Envía SNI (Server Name Indication) para que el servidor elija el certificado correcto |
| `-showcerts` | Muestra los certificados enviados por el servidor |
| `</dev/null` | Evita que el comando quede esperando entrada interactiva |
| `tee s_client.txt` | Guarda la salida para analizarla después |

El parámetro `-servername` merece atención. SNI (Server Name Indication) es la extensión de TLS que envía el hostname deseado justo al inicio de la conexión. Como varios dominios pueden compartir la misma dirección IP (Internet Protocol), el servidor usa esa información para decidir qué certificado debe presentar.

El archivo `s_client.txt` contiene los certificados en formato PEM (Privacy-Enhanced Mail), una representación textual de los datos binarios del certificado. Para separar cada certificado en un archivo propio, uso:

```bash
awk '
/-----BEGIN CERTIFICATE-----/ {n++; file=sprintf("cert-%02d.pem", n)}
file {print > file}
/-----END CERTIFICATE-----/ {close(file); file=""}
' s_client.txt
```

El resultado fue:

```text
cert-01.pem
cert-02.pem
cert-03.pem
```

Hasta aquí, lo único que sé es que *el servidor envió tres certificados*.

Todavía no sé cuál es el leaf, cuál es el intermediate y si alguno de ellos es una root.

Antes de continuar, vale la pena fijar la terminología. Leaf es el certificado del servidor final, el más cercano al dominio accedido. Intermediate es una CA intermedia, usada para firmar certificados leaf. Root es la CA en la cima de la jerarquía, normalmente usada como trust anchor.

## Pregunta 2: ¿cómo descubro el rol de cada certificado?

Inspecciono `subject`, `issuer`, `serial` y vigencia de cada uno:

```bash
for cert in cert-*.pem; do
  echo "===== $cert ====="
  openssl x509 -in "$cert" -noout \
    -subject \
    -issuer \
    -serial \
    -dates
  echo
done
```

La salida observada fue:

```text
===== cert-01.pem =====
subject=CN=www.mercadopago.com.br
issuer=C=US, O=Amazon, CN=Amazon RSA 2048 M04

===== cert-02.pem =====
subject=C=US, O=Amazon, CN=Amazon RSA 2048 M04
issuer=C=US, O=Amazon, CN=Amazon Root CA 1

===== cert-03.pem =====
subject=C=US, O=Amazon, CN=Amazon Root CA 1
issuer=C=US, ST=Arizona, L=Scottsdale, O=Starfield Technologies, Inc., CN=Starfield Services Root Certificate Authority - G2
```

Con esto ya puedo armar una primera lectura:

| Archivo | Rol probable | Por qué |
|---|---|---|
| `cert-01.pem` | Leaf | El `subject` es el dominio final |
| `cert-02.pem` | Intermediate | El `subject` es la CA que firmó el leaf |
| `cert-03.pem` | Certificado de Amazon Root CA 1 | El `subject` es `Amazon Root CA 1` |

En los campos anteriores, `subject` identifica la entidad representada por el certificado e `issuer` identifica quién lo emitió. El atributo `CN` (Common Name) aparece dentro de esos nombres y, históricamente, se usó para indicar el nombre principal asociado al certificado.

Pero fíjate en la palabra "probable". Ver el `subject` y el `issuer` ayuda a entender la cadena, pero no es la prueba criptográfica en sí misma.

El campo `issuer` es solo una declaración dentro del certificado. Lo que realmente prueba la relación entre dos certificados es la firma digital.

En otras palabras:

*Cert-01 dice que fue emitido por Amazon RSA 2048 M04, pero el cliente todavía necesita verificar si la firma realmente fue hecha con la clave privada de esa CA.*

## Pregunta 3: ¿cómo sé que el leaf es válido para el dominio accedido?

El certificado leaf no se acepta solo porque está al inicio de la cadena. Necesita representar el hostname correcto.

Verifico el SAN (Subject Alternative Name) del leaf:

```bash
openssl x509 -in cert-01.pem -noout -ext subjectAltName
```

Salida:

```text
X509v3 Subject Alternative Name:
    DNS:www.mercadopago.com.br, DNS:mercadopago.com.br
```

Esto responde una pregunta importante:

*¿El certificado leaf cubre el hostname que accedí?*

Sí.

SAN es la extensión del certificado que lista los nombres de dominio para los cuales es válido. El prefijo `DNS:` en la salida indica que esos valores son nombres de dominio aceptados por el certificado. Como accedí a `mercadopago.com.br`, el cliente espera encontrar ese hostname en el SAN del leaf. Si el certificado hubiera sido emitido para otro dominio, la conexión debería rechazarse, aunque toda la cadena de firmas fuera válida.

Históricamente, el campo `CN` (Common Name) también se usaba para indicar el dominio principal. En la validación moderna, sin embargo, el cliente debe comparar el hostname con el SAN, no solo con el CN.

También existen certificados con wildcard, como `*.mercadopago.com.br`. En ese caso, el wildcard cubre un único nivel a la izquierda, como `api.mercadopago.com.br`, pero no cubre algo como `a.b.mercadopago.com.br`. Las reglas de matching de hostname son más estrictas que simplemente buscar un sufijo parecido.

Este es un punto importante: la autenticación TLS no es solo "la cadena fue firmada por una CA confiable". También es:

*¿Esa cadena confiable representa el hostname que pedí?*

## Pregunta 4: ¿cómo nace una cadena como esta?

Antes de continuar con la PoC, vale la pena entender cómo se crearon estos certificados.

Una autoridad certificadora, o CA, comienza generando un par de claves: una clave privada de la CA y una clave pública de la CA.

La clave privada se usa para firmar certificados. La clave pública se distribuye dentro de los certificados para que los clientes puedan verificar esas firmas.

En la cima de la jerarquía, una root CA, es decir, la autoridad certificadora raíz, crea un certificado para sí misma:

```text
subject = Root CA
issuer  = Root CA
```

Este es un certificado self-signed, es decir, firmado por la propia clave privada de la CA que representa.

Después, una CA intermedia genera su propio par de claves y crea un CSR (Certificate Signing Request) que contiene la clave pública de la propia CA intermedia. La root firma ese certificado intermedio.

Por último, el servidor genera su par de claves, crea un CSR para el dominio deseado que contiene la clave pública del propio servidor, y la CA intermedia firma el certificado leaf.

Un CSR no es un certificado terminado. Es una solicitud de emisión que normalmente lleva:

- la clave pública del solicitante, que entrará en el certificado;
- el subject deseado;
- los SANs deseados;
- una firma hecha con la clave privada correspondiente, para probar que el solicitante realmente posee esa clave.

En el caso de un certificado público para dominio, la CA todavía necesita validar el control de ese dominio antes de emitir el leaf. La CA no coloca su propia clave pública en el leaf; coloca la clave pública del servidor y firma el certificado con la clave privada de la CA.

El esquema queda así:

**Root CA → firma → Intermediate CA → firma → Leaf certificate**

En el caso de Mercado Pago:

**Amazon Root CA 1 → firma → Amazon RSA 2048 M04 → firma → www.mercadopago.com.br**

En términos criptográficos:

- firma del leaf → hecha con la clave privada del intermediate
- firma del intermediate → hecha con la clave privada de la root

En la validación, el cliente hace el camino inverso:

- usa la clave pública del intermediate para validar el leaf
- usa la clave pública de la root para validar el intermediate

Pero la validación no se detiene en las firmas. El cliente también necesita verificar:

- si el hostname accedido aparece en el SAN del leaf;
- si los certificados están dentro del período de vigencia;
- si los certificados de CA tienen extensiones compatibles con el rol de CA;
- si la cadena respeta las restricciones de path building, es decir, si cada certificado puede ocupar ese rol en la cadena y firmar el siguiente;
- si el camino termina en una trust anchor aceptada localmente.

Vuelvo a estas reglas después de resolver la pregunta principal de la PoC: ¿qué camino eligió realmente OpenSSL para validar esta cadena?

## Pregunta 5: ¿el servidor envió la root final?

A primera vista, `cert-03.pem` parece una root porque su `subject` es `Amazon Root CA 1`.

Pero no es una root self-signed.

Mira de nuevo:

```text
subject=C=US, O=Amazon, CN=Amazon Root CA 1
issuer=C=US, ST=Arizona, L=Scottsdale, O=Starfield Technologies, Inc., CN=Starfield Services Root Certificate Authority - G2
```

Si fuera self-signed, esperaría:

```text
subject = Amazon Root CA 1
issuer  = Amazon Root CA 1
```

Pero el `issuer` es Starfield.

Esto ya plantea una nueva pregunta:

*¿Cómo puede un certificado representar a Amazon Root CA 1 y, al mismo tiempo, estar firmado por Starfield?*

Pero, por ahora, la respuesta a la pregunta actual ya está en el `issuer`: el último certificado enviado por el servidor todavía apunta a otra CA. Entonces no es la root self-signed final.

La respuesta es:

*El servidor no envió la root final que OpenSSL usó como trust anchor, es decir, como el punto final ya confiado localmente para cerrar la validación.*

## Pregunta 6: si el servidor no envió la root final, ¿cómo pasó la validación?

Ahora junto los certificados auxiliares enviados por el servidor:

```bash
cat cert-02.pem cert-03.pem > intermediates.pem
```

Y le pido a OpenSSL que valide el leaf:

```bash
openssl verify \
  -show_chain \
  -verify_hostname mercadopago.com.br \
  -untrusted intermediates.pem \
  cert-01.pem
```

Salida:

```text
cert-01.pem: OK
Chain:
depth=0: CN=www.mercadopago.com.br (untrusted)
depth=1: C=US, O=Amazon, CN=Amazon RSA 2048 M04 (untrusted)
depth=2: C=US, O=Amazon, CN=Amazon Root CA 1
```

Esta salida es extremadamente interesante.

El leaf y el intermediate aparecen como `untrusted` porque fueron entregados a OpenSSL como material auxiliar para construir la cadena, y no como trust anchors. En este experimento, ese material vino del servidor. Eso no significa que sean maliciosos.

Pero el `depth=2`, `Amazon Root CA 1`, no aparece como `untrusted`.

Eso indica que OpenSSL encontró localmente una `Amazon Root CA 1` confiable y cerró la cadena ahí.

En otras palabras:

El servidor envió: **Leaf → Amazon RSA 2048 M04 → Amazon Root CA 1 cross-signed**.

OpenSSL validó: **Leaf → Amazon RSA 2048 M04 → Amazon Root CA 1 local**.

Todavía necesito explicar con calma qué significa `cross-signed`. Por ahora, el punto importante es que el certificado enviado por el servidor no fue el certificado que cerró la validación.

Esta es una de las ideas más importantes de todo el artículo:

*La cadena presentada por el servidor no tiene que ser idéntica a la cadena elegida por el cliente.*

El servidor entrega material para que el cliente construya un camino.

El cliente elige el camino que consigue validar hasta una trust anchor local.

## Pregunta 7: ¿dónde vivía esa trust anchor local?

Al comienzo de la PoC, el comando `openssl version -a` ya había mostrado que el OpenSSL instalado vía Homebrew usaba `OPENSSLDIR: "/opt/homebrew/etc/openssl@3"`.

Ahora esa pista pasa a importar. Dentro de ese directorio, el bundle de CAs confiables usado por este OpenSSL estaba en `/opt/homebrew/etc/openssl@3/cert.pem`.

Hago la validación explícita usando ese bundle directamente:

```bash
openssl verify \
  -no-CApath \
  -no-CAstore \
  -CAfile /opt/homebrew/etc/openssl@3/cert.pem \
  -show_chain \
  -verify_hostname mercadopago.com.br \
  -untrusted intermediates.pem \
  cert-01.pem
```

El resultado siguió siendo `OK`, confirmando que la trust anchor usada por OpenSSL venía de ese bundle local.

Este punto es importante porque no existe un trust store universal.

Vale la pena separar los dos términos. Trust store es el conjunto local de CAs confiables disponible para el cliente. Trust anchor es el certificado o la clave específica de ese conjunto que cierra una validación concreta.

| Entorno | Trust store principal |
|---|---|
| macOS nativo | Keychain |
| OpenSSL de Homebrew | Bundle propio en `cert.pem` |
| Android | System store, user store y anchors de la aplicación |
| Firefox | Store propio basado en NSS |
| Java | `cacerts` |

Una cadena puede ser válida en el navegador e inválida en OpenSSL. Puede ser válida en macOS e inválida en una JVM. Puede ser aceptada por el Android de sistema y rechazada por una aplicación que incorpora sus propias anchors.

La trust anchor siempre es una decisión local del cliente.

También vale la pena guardar un matiz importante: trust anchor es un rol dentro de la validación, no solo un tipo de certificado. La `Amazon Root CA 1` self-signed local actuó como trust anchor en el camino que observé. En cambio, la `Amazon Root CA 1` cross-signed enviada por el servidor, en el camino legado, actuaría como un certificado intermedio adicional hasta Starfield.

## Pregunta 8: ¿cómo demostrar que había una Amazon Root CA 1 local?

Extraigo los certificados del bundle de OpenSSL:

```bash
mkdir -p roots

awk '
/-----BEGIN CERTIFICATE-----/ {n++; file=sprintf("roots/root-%03d.pem", n)}
file {print > file}
/-----END CERTIFICATE-----/ {close(file); file=""}
' /opt/homebrew/etc/openssl@3/cert.pem
```

Después busco la `Amazon Root CA 1`:

```bash
for cert in roots/root-*.pem; do
  subject=$(openssl x509 -in "$cert" -noout -subject)
  if echo "$subject" | grep -q "Amazon Root CA 1"; then
    echo "$cert"
    openssl x509 -in "$cert" -noout -subject -issuer -serial -dates
  fi
done
```

Encuentro:

```text
roots/root-022.pem
subject=C=US, O=Amazon, CN=Amazon Root CA 1
issuer=C=US, O=Amazon, CN=Amazon Root CA 1
```

Esa era la versión self-signed local de `Amazon Root CA 1`.

Aquí aparece una distinción importante:

*Una root no es confiable porque sea self-signed; es confiable porque fue agregada al trust store del cliente.*

Cualquier persona puede generar un certificado self-signed. Lo que convierte una root en trust anchor es la decisión local del sistema, runtime o aplicación de confiar en ella.

## Pregunta 9: ¿cómo puede existir Amazon Root CA 1 en dos certificados diferentes?

Ahora llego al punto más sutil de la PoC.

Comparo el certificado enviado por el servidor con la root local:

```bash
echo "===== cert-03.pem enviado por el servidor ====="
openssl x509 -in cert-03.pem -noout -subject -issuer -fingerprint -sha256

echo
echo "===== root local ====="
openssl x509 -in roots/root-022.pem -noout -subject -issuer -fingerprint -sha256
```

El resultado conceptual fue:

| Campo | `cert-03.pem` | `roots/root-022.pem` |
|---|---|---|
| Subject | Amazon Root CA 1 | Amazon Root CA 1 |
| Issuer | Starfield Services Root Certificate Authority - G2 | Amazon Root CA 1 |
| Fingerprint | Diferente | Diferente |

Esto prueba que son certificados diferentes.

Fingerprint es el hash del certificado completo. Si cualquier detalle del certificado cambia, como issuer, número de serie, vigencia o firma, el fingerprint también cambia.

Pero todavía faltaba responder:

*Si son certificados diferentes, ¿por qué ambos sirven para validar la misma CA?*

## Pregunta 10: ¿qué es igual entre ellos?

Comparo el hash del SPKI (Subject Public Key Info) de ambos:

```bash
echo "===== hash del SPKI cert-03.pem ====="
openssl x509 -in cert-03.pem -pubkey -noout \
  | openssl pkey -pubin -outform DER \
  | openssl dgst -sha256

echo "===== hash del SPKI root local ====="
openssl x509 -in roots/root-022.pem -pubkey -noout \
  | openssl pkey -pubin -outform DER \
  | openssl dgst -sha256
```

Salida:

```text
SHA2-256(stdin)= fbe3018031f9586bcbf41727e417b7d1c45c2f47f93be372a17b96b50757d5a2
SHA2-256(stdin)= fbe3018031f9586bcbf41727e417b7d1c45c2f47f93be372a17b96b50757d5a2
```

El fingerprint del certificado completo era diferente.

El hash del SPKI era igual.

Esto significa que los dos certificados llevaban la misma clave pública.

SPKI es la estructura del certificado que describe la clave pública: qué algoritmo usa y cuáles son los bytes de la clave en sí.

Esa es la diferencia fundamental:

- **Fingerprint**: hash del certificado completo
- **Hash del SPKI**: hash de la estructura de la clave pública

Si cualquier campo del certificado cambia, el fingerprint cambia:

- issuer;
- número de serie;
- vigencia;
- extensiones;
- firma.

Pero si la clave pública sigue siendo la misma, el SPKI sigue siendo igual.

## Pregunta 11: ¿qué significa cross-signed, al final?

Una root self-signed de Amazon Root CA 1 tiene:

```text
subject = Amazon Root CA 1
issuer  = Amazon Root CA 1
```

Es un certificado emitido por la propia Amazon Root CA 1 para su propia clave pública.

La versión cross-signed, en cambio, tiene:

```text
subject = Amazon Root CA 1
issuer  = Starfield Services Root Certificate Authority - G2
```

Lleva la misma clave pública de Amazon Root CA 1, pero fue firmada por Starfield.

El proceso conceptual es:

1. Amazon genera el par de claves de Amazon Root CA 1.
2. Amazon crea un certificado self-signed para esa clave.
3. Para obtener una versión cross-signed, Amazon prepara una solicitud de certificación para Starfield, normalmente representada por un CSR.
4. Ese CSR lleva la clave pública de Amazon Root CA 1 y está firmado con la clave privada correspondiente, probando que Amazon controla esa clave.
5. Starfield usa esa solicitud para emitir otro certificado para la misma clave pública de Amazon Root CA 1.
6. Ese segundo certificado es firmado por Starfield.

Aquí vale un cuidado de lenguaje: no es el certificado el que "genera" el CSR. Es la entidad operadora de la CA, en este caso Amazon, la que crea esa solicitud usando la clave privada asociada a Amazon Root CA 1. El punto conceptual importante es que Starfield nunca recibe la clave privada de Amazon; recibe una solicitud que contiene la clave pública y una prueba de posesión de la clave privada correspondiente.

Así surgen dos certificados diferentes: la **Amazon Root CA 1 self-signed** y la **Amazon Root CA 1 cross-signed**.

En este caso, representan la misma CA lógica porque tienen el mismo subject, llevan la misma clave pública y ambos son certificados de CA.

Son certificados diferentes porque fueron emitidos en contextos diferentes y firmados por autoridades diferentes.

## Pregunta 12: ¿por qué alguien haría esto?

Por compatibilidad.

Cuando surgió Amazon Root CA 1, los clientes modernos podían recibir esa root en sus trust stores y confiar en ella directamente.

Pero los clientes antiguos tal vez todavía no tenían esa root instalada.

Esos clientes antiguos ya confiaban en roots más viejas, como Starfield.

Entonces el cross-signing crea un puente:

Cliente moderno: **Leaf → Amazon RSA 2048 M04 → Amazon Root CA 1 local**.

Cliente legado: **Leaf → Amazon RSA 2048 M04 → Amazon Root CA 1 cross-signed → Starfield Root**.

En el cliente moderno, la validación termina antes, porque ya conoce Amazon Root CA 1.

En el cliente legado, la validación continúa hasta Starfield, porque es en ella en quien confía localmente.

## Pregunta 13: ¿por qué el cliente moderno consigue cerrar la cadena en la root local en lugar de continuar por la versión cross-signed?

Porque la firma del intermediate fue hecha con la clave privada de Amazon Root CA 1.

Para verificar esa firma, el cliente necesita la clave pública correspondiente.

Tanto la versión self-signed como la versión cross-signed llevan la misma clave pública.

Entonces:

**clave pública de Amazon Root CA 1 self-signed = clave pública de Amazon Root CA 1 cross-signed**

Por eso ambas logran validar la firma de `Amazon RSA 2048 M04`.

El cliente no sustituye literalmente un certificado por otro. El path builder encuentra una candidata local confiable con el mismo subject y la misma clave pública, y construye un camino alternativo hasta ella.

El cliente moderno no necesita continuar hasta Starfield porque ya posee localmente una trust anchor de Amazon Root CA 1 que lleva esa misma clave pública.

El cliente legado, en cambio, usa el certificado cross-signed como un intermediario adicional:

1. Valida el leaf con la clave pública de Amazon RSA 2048 M04.
2. Valida Amazon RSA 2048 M04 con la clave pública de Amazon Root CA 1.
3. Valida la Amazon Root CA 1 cross-signed con la clave pública de Starfield.
4. Cierra la cadena en la Starfield local.

## Una última capa: ¿qué más valida el cliente?

Hasta aquí, resolví el misterio central de la PoC: el servidor envió una cadena, pero OpenSSL construyó otra hasta una trust anchor local compatible.

Aun así, vale la pena volver a una observación que quedó abierta en la Pregunta 4. Una cadena no se acepta solo porque las firmas coinciden. El cliente también verifica si cada certificado puede ocupar el rol que está ocupando.

Una de las extensiones más importantes para esto es `basicConstraints`.

Si quiero inspeccionarla en los certificados capturados, puedo usar:

```bash
openssl x509 -in cert-01.pem -noout -ext basicConstraints
openssl x509 -in cert-02.pem -noout -ext basicConstraints
```

Lo que busco en esa salida es la diferencia entre `CA:FALSE` y `CA:TRUE`.

Un leaf normalmente aparece con `CA:FALSE`. Esto significa que puede representar a un servidor, pero no puede ser aceptado como una CA capaz de firmar otro certificado por debajo de él en una cadena válida.

Un intermediate normalmente aparece con `CA:TRUE`. Esto significa que puede ser aceptado como autoridad certificadora y firmar certificados leaf.

Esta distinción impide que alguien tome un certificado común de servidor e intente usarlo como si fuera una CA intermedia. La clave privada correspondiente todavía podría producir una firma criptográficamente válida, pero la cadena debería rechazarse porque ese certificado no tiene permiso para actuar como CA.

Además, una CA puede llevar una restricción como `pathlen:0`. En ese caso, puede firmar certificados leaf, pero no puede firmar otro intermediate por debajo de ella. Esta es una forma de limitar la profundidad de la cadena e impedir que la jerarquía crezca indefinidamente.

Otra extensión relacionada es `keyUsage`. Aunque un certificado tenga `CA:TRUE`, normalmente también necesita permitir la firma de certificados para actuar realmente como CA.

Por último, este artículo no cubre los mecanismos de revocación, como CRL (Certificate Revocation List) y OCSP (Online Certificate Status Protocol). Forman otra parte importante de la validación en algunos clientes, pero no eran necesarios para responder la pregunta central de esta PoC.

## Lo que el experimento realmente me enseñó

Al final de la PoC, logré probar:

1. el servidor envió un leaf, un intermediate y un certificado auxiliar;
2. el leaf era válido para `mercadopago.com.br` porque el SAN contenía ese hostname;
3. el certificado auxiliar enviado por el servidor era una versión cross-signed de Amazon Root CA 1;
4. OpenSSL no cerró la validación en ese certificado enviado por el servidor;
5. OpenSSL encontró localmente una Amazon Root CA 1 self-signed;
6. la versión local y la versión cross-signed tenían fingerprints diferentes;
7. ambas tenían el mismo hash de SPKI;
8. por lo tanto, eran certificados diferentes que representaban la misma CA lógica.

## Modelo mental final

Después de recorrer todo el experimento, la forma más útil de pensar en TLS es:

1. El servidor presenta certificados.
2. El cliente verifica el hostname en el SAN del leaf.
3. El cliente valida firma por firma.
4. El cliente intenta construir un camino hasta una trust anchor local.
5. La cadena usada por el cliente puede ser diferente de la cadena presentada por el servidor.

Y la idea más importante de todas:

**Un certificado no es lo mismo que una clave pública.**

Un certificado es un documento firmado.

Una clave pública es el material criptográfico que ese documento asocia a una entidad.

Por eso, dos certificados pueden:

- tener issuers diferentes;
- tener números de serie diferentes;
- tener fingerprints diferentes;
- y, cuando conservan la misma clave pública y siguen siendo certificados de CA, aun así representar la misma autoridad certificadora.

En TLS, confiar en un dominio no significa confiar en un archivo aislado. Significa verificar que el hostname solicitado aparece en el leaf y confiar en un camino de firmas que termina en una trust anchor ya aceptada localmente por el cliente.

## Referencia rápida de los términos usados

| Término | Significado |
|---|---|
| HTTPS (Hypertext Transfer Protocol Secure) | HTTP transportado sobre una conexión TLS |
| Handshake | Negociación inicial en la que cliente y servidor acuerdan parámetros y validan identidad |
| TLS (Transport Layer Security) | Protocolo que protege la confidencialidad, integridad y autenticidad de la conexión |
| PKI (Public Key Infrastructure) | Infraestructura que organiza claves, certificados y autoridades certificadoras |
| CA (Certificate Authority) | Autoridad certificadora que firma certificados |
| SNI (Server Name Indication) | Extensión de TLS que informa al servidor a qué hostname quiere acceder el cliente |
| PEM (Privacy-Enhanced Mail) | Formato textual usado para representar certificados y claves |
| Leaf | Certificado del servidor final, el más cercano al dominio accedido |
| Intermediate | CA intermedia usada para firmar certificados leaf |
| Root CA | CA en la cima de la jerarquía de confianza |
| CN (Common Name) | Atributo histórico de nombre dentro del certificado |
| SAN (Subject Alternative Name) | Extensión que lista los hostnames válidos para el certificado |
| CSR (Certificate Signing Request) | Solicitud de emisión que lleva la clave pública del solicitante y los atributos deseados para el certificado |
| basicConstraints | Extensión que indica si un certificado puede actuar como CA y, cuando está presente, limita cuántas CAs adicionales pueden existir por debajo de él |
| keyUsage | Extensión que indica para qué operaciones criptográficas puede usarse la clave del certificado |
| CRL (Certificate Revocation List) | Lista publicada por una CA con certificados revocados |
| OCSP (Online Certificate Status Protocol) | Protocolo para consultar el estado de revocación de un certificado |
| Trust store | Conjunto local de CAs confiables disponible para el cliente |
| Trust anchor | Certificado o clave confiable que cierra una validación concreta |
| SPKI (Subject Public Key Info) | Estructura del certificado que describe la clave pública |
| Fingerprint | Hash del certificado completo |
| pathlen | Restricción en un certificado de CA que limita cuántas CAs intermedias adicionales pueden existir por debajo de él en la cadena |
| Path building | Proceso por el cual el cliente construye un camino de certificados, desde el leaf hasta una trust anchor local, a partir del material disponible |
| Self-signed | Certificado firmado por la propia clave privada de la entidad que representa |
| Cross-signed | Certificado de una CA firmado por otra CA |
