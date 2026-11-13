+++
title = "Por que instalar a CA do Burp não basta para interceptar um app Android"
translationKey = "android-burp-ca-trust-vs-pinning"
date = "2026-11-09"
author = "Thiago Siqueira"
tags = ["android", "ssl-pinning", "mobile", "burp", "network-security-config"]
keywords = ["android", "network security config", "certificate pinning", "burp suite", "trust store"]
description = "Um laboratório Android com múltiplas flavors para separar, passo a passo, uma falha comum de trust store de uma falha real de certificate pinning via Network Security Config, usando o Burp Suite como proxy MITM."
showFullContent = false
readingTime = true
hideComments = false
+++

Eu já tenho a CA (Certificate Authority) do Burp instalada no Android.

O Chrome consegue abrir a API (Application Programming Interface) do laboratório.

O Burp consegue interceptar a conexão HTTPS (Hypertext Transfer Protocol Secure) e mostrar a requisição no histórico.

Então por que uma app Android continua falhando?

Essa é a pergunta que abre a segunda parte do meu laboratório de SSL pinning.

No artigo anterior, eu quis entender como um cliente valida a autenticidade de um domínio durante o handshake TLS (Transport Layer Security). Eu percorri leaf, intermediate, trust anchor, SAN (Subject Alternative Name), SPKI (Subject Public Key Info) e cross-signing até chegar a um modelo mental importante: *uma conexão TLS só é aceita quando o cliente consegue construir um caminho até uma trust anchor que ele já confia localmente.*

Esse modelo já responde em quais CAs a app confia. Mas há uma pergunta adicional que ele não cobre: depois de confiar na CA, a app confia em qualquer chave pública que essa CA assine? É essa pergunta que dá nome a certificate pinning, também chamado de SSL pinning: a técnica pela qual uma aplicação, além de validar a cadeia até uma CA confiável, passa a exigir que a chave pública apresentada pelo servidor corresponda a um conjunto específico de chaves esperadas, os pins. É essa técnica que eu quero investigar no restante da série, e este segundo artigo começa isolando a parte mais confundida dela: separar uma falha comum de trust store de uma falha real de pinning.

Para responder a essa pergunta sem depender de um serviço real, eu montei um laboratório pequeno e controlado.

## Antes de começar: o laboratório do experimento

Eu não queria estudar pinning contra uma API real, porque isso misturaria meu experimento com certificados, chaves, rotação e disponibilidade de um serviço que eu não controlo.

Então eu construí uma API HTTPS local, pequena e descartável. Ela existe apenas para servir como alvo previsível do laboratório:

- eu controlo o hostname;
- eu controlo o certificado leaf, isto é, o certificado final apresentado pelo servidor;
- eu controlo a Root CA que assina esse certificado;
- eu consigo calcular exatamente o SPKI, isto é, a estrutura da chave pública cujo hash a app deve aceitar;
- eu consigo mudar a chave depois, quando quiser estudar rotação.

Ela expõe um endpoint `/health`, usado para validar conectividade e TLS, e um endpoint `/profile`, consultado pela app para receber um payload fictício.

Também criei uma app Android mínima. Ela tem uma única tela, um botão para disparar a requisição e uma área onde eu consigo ver `SUCCESS`, `FAILURE`, o status HTTP ou a exceção capturada. Ela não tenta simular uma aplicação de negócio real; ela existe para tornar visível em qual etapa a conexão é aceita ou rejeitada.

O ponto mais importante é que eu não criei várias apps com lógicas diferentes. Eu criei a mesma app com várias flavors, isto é, variantes de build. Todas usam a mesma `MainActivity`, chamam a mesma URL `/profile` e exibem a mesma interface. O que muda entre elas é apenas a política de confiança. Isso foi importante porque me permitiu comparar uma variável por vez. Se duas flavors chamam a mesma API, pelo mesmo caminho, e só uma falha, a diferença não está no backend nem na rede: está na política de confiança daquela variante.

A arquitetura do laboratório ficou assim: **Android Emulator → (HTTPS) → Burp Suite no macOS → (HTTPS) → API local em 127.0.0.1:8443**.

O Android Emulator representa o device analisado. O Burp Suite roda no macOS e fica no meio da conexão como proxy MITM (Man-in-the-Middle). A API local também roda no macOS, mas em outra porta, servindo o certificado e o payload que eu controlo.

O endpoint de conectividade ficava em `https://api.ssl-pinning.127.0.0.1.nip.io:8443/health`.

Na prática, eu alternava entre dois caminhos:

- **Android Emulator → API local**
- **Android Emulator → Burp Suite → API local**

## A primeira evidência

Com o laboratório montado, eu quero reproduzir uma situação comum em análise mobile:

1. configurar o Burp Suite como proxy;
2. instalar a CA do Burp no Android;
3. confirmar que o navegador aceita a conexão;
4. abrir uma app Android;
5. observar que ela ainda falha;
6. descobrir se a falha vem de trust store ou de certificate pinning.

O fio condutor do artigo passa a ser:

- *O que a app está rejeitando?*
- *Uma CA que ela não confia?*
- *Ou uma chave pública que não bate com o pin esperado?*

Antes de testar qualquer app, eu valido os dois caminhos no Chrome do Android.

Sem proxy:

```bash
adb shell settings put global http_proxy :0
adb shell am start -a android.intent.action.VIEW \
  -d 'https://api.ssl-pinning.127.0.0.1.nip.io:8443/health?mode=android-direct'
```

Com proxy:

```bash
adb shell settings put global http_proxy 10.0.2.2:8080
adb shell am start -a android.intent.action.VIEW \
  -d 'https://api.ssl-pinning.127.0.0.1.nip.io:8443/health?mode=android-burp'
```

Os comandos acima fazem duas coisas diferentes:

| Trecho | Função |
| --- | --- |
| `settings put global http_proxy :0` | Remove o proxy global do Android |
| `settings put global http_proxy 10.0.2.2:8080` | Faz o emulador enviar tráfego para o Burp no host |
| `10.0.2.2` | Alias especial que, dentro do emulador, aponta para o macOS host |
| `am start -a android.intent.action.VIEW -d <url>` | Pede ao Android para abrir a URL no navegador |

Eu uso o hostname `127.0.0.1.nip.io` porque preciso que o mesmo nome DNS funcione tanto no emulador quanto no Burp, mantendo a validação de hostname do TLS.

No Android, a ponte até a API local depende de uma porta encaminhada do host: `adb reverse tcp:8443 tcp:8443` redireciona a porta 8443 do emulador para a mesma porta no macOS, onde a API local está escutando.

Nos dois casos, o Chrome mostra o JSON de saúde da API:

```json
{
  "status": "ok",
  "service": "ssl-pinning-lab-api",
  "time": "..."
}
```

No segundo caso, a requisição também aparece no HTTP history do Burp.

<img src="/images/posts/02-por-que-instalar-a-ca-do-burp-nao-basta-para-interceptar-um-app-android/chrome-and-burp-health.png" alt="Chrome abrindo o endpoint health enquanto o Burp registra a mesma requisição" class="center" width="820">

*O Chrome aceita a conexão interceptada e a requisição aparece no histórico do Burp.*

Para interpretar esse teste, há um detalhe importante: no conjunto de certificados instalados pelo usuário no Android, que mais adiante vou chamar de user store, eu já havia instalado duas CAs diferentes:

- a Root CA do laboratório, que assina o certificado servido diretamente pela API;
- a CA do Burp, que assina os certificados gerados dinamicamente durante o MITM.

Quando eu acesso a API diretamente, o Android precisa confiar na Root CA do laboratório. Quando eu passo pelo Burp, o Android deixa de ver o certificado original da API e passa a ver um certificado emitido pela CA do Burp.

Isso prova cinco coisas:

1. a API está acessível;
2. o hostname está correto;
3. o Chrome aceita a Root CA do laboratório no caminho direto;
4. o Chrome aceita a CA do Burp no caminho interceptado;
5. o proxy consegue interceptar o HTTPS do Chrome no device.

Mas isso ainda prova apenas o comportamento do Chrome e da infraestrutura. A app Android ainda pode tomar uma decisão de confiança diferente.

Então eu passo para a app.

## Antes de testar as flavors: onde a política de confiança entra?

Como todas as flavors usam a mesma `MainActivity` e chamam a mesma URL `/profile`, eu preciso encontrar onde a política de confiança realmente entra na app.

Antes de rodar a primeira variante, eu procuro essa declaração no `AndroidManifest.xml`. É ali que encontro a referência para um arquivo chamado `network_security_config`:

```xml
<application
    android:networkSecurityConfig="@xml/network_security_config"
    ...>
```

Network Security Config, ou NSC, é o mecanismo declarativo do Android para definir trust anchors, regras de cleartext, pins e exceções de debug.

O manifest é comum a todas as flavors. O que muda no build é o recurso que ocupa esse caminho: `app/src/<flavor>/res/xml/network_security_config.xml`.

Ou seja, o código Java continua abrindo a mesma URL, mas o Android passa a consultar uma política de confiança diferente conforme a flavor instalada.

O Chrome usa `/health` apenas para validar a infraestrutura. A partir daqui, as flavors da app chamam `/profile`, sempre pelo mesmo caminho, mudando apenas o XML de confiança.

Isso define a comparação que eu faço dali em diante:

- mesma tela;
- mesma URL;
- mesmo Burp;
- mesma API;
- **XML de confiança diferente**.

## Pergunta 1: o que acontece com uma flavor que confia apenas em CAs do sistema?

Eu começo pela variante `systemOnly`.

Ela usa `HttpsURLConnection`, o cliente HTTPS nativo da plataforma Java/Android, e uma configuração de rede mínima:

```xml
<network-security-config>
    <base-config cleartextTrafficPermitted="false">
        <trust-anchors>
            <certificates src="system" />
        </trust-anchors>
    </base-config>
</network-security-config>
```

O nome já dá uma pista: essa flavor confia apenas em CAs do sistema.

Eu a executo com o Burp ainda configurado como proxy:

```bash
adb shell am force-stop com.example.sslpinninglab.systemonly
adb shell am start \
  -n com.example.sslpinninglab.systemonly/com.example.sslpinninglab.MainActivity \
  --ez auto_request true
```

Esse padrão de comando se repete nas próximas flavors:

| Trecho | Função |
| --- | --- |
| `am force-stop <package>` | Encerra a instância anterior para o teste começar limpo |
| `am start -n <package>/<activity>` | Abre exatamente a flavor desejada |
| `--ez auto_request true` | Envia um boolean extra para a app disparar a requisição ao abrir |

A tela mostra `FAILURE`, com `Exception: SSLHandshakeException` e causa raiz `CertPathValidatorException: Trust anchor for certification path not found.`

<img src="/images/posts/02-por-que-instalar-a-ca-do-burp-nao-basta-para-interceptar-um-app-android/system-only-burp-failure.png" alt="Flavor systemOnly falhando via Burp com CertPathValidatorException" class="center" width="420">

*`systemOnly` falha via Burp antes de qualquer comparação de pins.*

O logcat, isto é, o stream de logs do Android, confirma a mesma causa: `Caused by: java.security.cert.CertPathValidatorException: Trust anchor for certification path not found.`

Nesse momento, a primeira conclusão importante aparece: *essa falha ainda não é pinning.*

A app nem chega a comparar SPKIs.

Ela falha antes, porque a cadeia apresentada pelo Burp não termina em uma CA aceita por aquela variante.

Na verdade, `systemOnly` também falha sem Burp. A Root CA da própria API local também está instalada no user store, e essa flavor não aceita nenhuma CA vinda dali. Isso reforça o ponto: `systemOnly` não existe para provar pinning; ela existe para provar que uma app pode rejeitar a conexão antes mesmo de qualquer pin existir.

## Pergunta 2: por que a CA do Burp instalada no Android não entra na validação?

A CA do Burp está instalada no Android, mas ela está no user store, isto é, no conjunto de certificados adicionados pelo usuário.

Esse detalhe importa porque apps Android modernas não confiam em user-added CAs por padrão.

Em apps com `targetSdkVersion >= 24`, isto é, apps que declaram Android 7.0 ou superior como versão alvo, o comportamento padrão passa a ser confiar em CAs do sistema e não confiar automaticamente em CAs adicionadas pelo usuário.

No laboratório, `systemOnly` torna essa decisão explícita com `src="system"`. Já `proxyTrusted` faz o opt-in contrário, adicionando `src="user"` de forma deliberada.

Isso explica o aparente paradoxo:

| Cliente | O que ele aceita | Resultado |
| --- | --- | --- |
| Chrome Android | CA instalada pelo usuário | Passa |
| `systemOnly` | Apenas CAs do sistema | Falha |

A pergunta correta não é *a CA do Burp está instalada?* A pergunta correta é *essa app decidiu confiar em CAs adicionadas pelo usuário?*

No caso de `systemOnly`, a resposta é não.

Esse é um ponto importante do laboratório porque evita um erro comum: *ver uma falha de handshake e concluir imediatamente que existe pinning.*

Nesse caso, a falha vem apenas da política de trust anchors.

## Pergunta 3: o que muda quando a app passa a aceitar user-added CAs?

Para responder isso, eu rodo a variante `proxyTrusted`.

Ela chama o mesmo endpoint, usa o mesmo Burp e mantém o mesmo fluxo de rede. A única mudança está no Network Security Config:

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

Agora a app aceita:

- CAs do sistema;
- CAs adicionadas pelo usuário.

O atributo `overridePins="false"` torna explícita uma decisão que já seria o padrão fora de `debug-overrides`. Ele diz que uma CA vinda do user store pode participar da validação normal da cadeia, mas não ganha permissão para ignorar pins caso eles existam. Em `proxyTrusted`, ainda não há `<pin-set>`, então esse detalhe não muda o resultado. Ele documenta a intenção que vou manter na comparação com `nscPinned`; omitir o atributo aqui não mudaria seu valor padrão.

Eu executo:

```bash
adb shell am force-stop com.example.sslpinninglab.proxytrusted
adb shell am start \
  -n com.example.sslpinninglab.proxytrusted/com.example.sslpinninglab.MainActivity \
  --ez auto_request true
```

O resultado é `SUCCESS`, com `HTTP 200`.

Na tela, a app também mostra o payload devolvido pela API:

```json
{
  "user": "alice.demo",
  "accountTier": "lab",
  "message": "Fictional data for SSL pinning study only"
}
```

<img src="/images/posts/02-por-que-instalar-a-ca-do-burp-nao-basta-para-interceptar-um-app-android/proxy-trusted-burp-success.png" alt="Flavor proxyTrusted funcionando via Burp e retornando HTTP 200" class="center" width="420">

*`proxyTrusted` aceita a CA do Burp e consegue ler o payload da API.*

O logcat também registra: `proxy-trusted request succeeded`.

Essa variante é a baseline que eu preciso.

Ela prova que:

1. o Burp está funcionando;
2. a CA do Burp é confiável para aquela app;
3. a rota até a API está correta;
4. o handshake TLS pode terminar com sucesso;
5. qualquer falha posterior em outra variante já não pode ser atribuída simplesmente ao proxy.

Ela também passa no caminho direto, porque `src="user"` aceita a Root CA local da API. Em outras palavras, `proxyTrusted` é a primeira flavor em que tanto o caminho direto quanto o caminho via Burp funcionam.

Esse é o papel mais importante de `proxyTrusted` no laboratório.

Ela não existe para mostrar uma app “vulnerável”.

Ela existe para provar que a infraestrutura de interceptação está correta antes de falar em pinning.

## Pergunta 4: o que acontece quando eu adiciono um `<pin-set>` à mesma base de confiança?

Depois de validar `proxyTrusted`, eu rodo a variante `nscPinned` para responder exatamente essa pergunta.

Ela usa exatamente a mesma base de confiança:

```xml
<trust-anchors>
    <certificates src="system" />
    <certificates src="user" overridePins="false" />
</trust-anchors>
```

Mas adiciona um `pin-set` para o domínio da API:

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

Esses pins não precisam ser decorados ou inventados manualmente. Eles são o mesmo hash do SPKI que eu já havia aprendido a extrair no artigo anterior, só que em outro formato. Lá eu usei:

```bash
openssl x509 -in cert.pem -pubkey -noout \
  | openssl pkey -pubin -outform DER \
  | openssl dgst -sha256
```

para obter o hash do SPKI em hexadecimal. O Network Security Config espera esse mesmo hash em base64, então o comando ganha um passo extra:

```bash
openssl x509 -in cert.pem -pubkey -noout \
  | openssl pkey -pubin -outform DER \
  | openssl dgst -sha256 -binary \
  | openssl base64
```

A diferença é só a codificação: `-binary` evita a saída em hexadecimal, e `openssl base64` converte esses bytes para o formato que o atributo `<pin digest="SHA-256">` espera. O conceito continua o mesmo: é o hash do SPKI, não do certificado inteiro, que entra no pin.

Os dois valores do `<pin-set>` não significam que duas chaves estejam sendo usadas ao mesmo tempo pela API. O primeiro é o pin da chave atual do servidor. O segundo é um backup pin, reservado para uma futura rotação. Neste artigo, o ponto importante é apenas que o certificado gerado pelo Burp não bate com nenhum deles.

No caminho direto, sem proxy, `nscPinned` passa. A API local apresenta exatamente a chave cujo hash do SPKI está no XML. O teste que eu quero observar é o caminho via Burp, em que a cadeia ainda pode ser confiável, mas a chave pública apresentada muda.

Primeiro eu removo o proxy global do Android e executo a flavor diretamente contra a API:

```bash
adb shell settings put global http_proxy :0
adb shell am force-stop com.example.sslpinninglab.nscpinned
adb shell am start \
  -n com.example.sslpinninglab.nscpinned/com.example.sslpinninglab.MainActivity \
  --ez auto_request true
```

O resultado direto é `SUCCESS`, com `HTTP 200`.

Depois eu recoloco o Burp como proxy e repito exatamente a mesma flavor:

```bash
adb shell settings put global http_proxy 10.0.2.2:8080
adb shell am force-stop com.example.sslpinninglab.nscpinned
adb shell am start \
  -n com.example.sslpinninglab.nscpinned/com.example.sslpinninglab.MainActivity \
  --ez auto_request true
```

O resultado via Burp é `FAILURE`, com `Exception: SSLHandshakeException` e causa raiz `CertificateException: Pin verification failed`.

O logcat mostra o ponto exato da falha: `Caused by: java.security.cert.CertificateException: Pin verification failed`, em `NetworkSecurityTrustManager.checkPins(...)`.

Agora a situação é diferente de `systemOnly`.

Em `systemOnly`, a cadeia do Burp nem chega a uma trust anchor aceita.

Em `nscPinned`, a CA do Burp já é aceita.

O problema agora é outro: *a chave pública apresentada pelo Burp não bate com os SPKIs esperados pela app.*

Esse é o ponto em que o pinning realmente entra.

Sem pinning, a pergunta da app é: *essa cadeia termina em uma CA que eu confio?*

Com pinning, a pergunta passa a ser: *essa cadeia termina em uma CA que eu confio e também contém uma chave pública esperada?*

A comparação central do laboratório fica:

| Variante | User CA aceita? | Pinning? | Direto | Via Burp | Causa da falha via Burp |
| --- | --- | --- | --- | --- | --- |
| `systemOnly` | Não | Não | Falha | Falha | Trust anchor não aceita |
| `proxyTrusted` | Sim | Não | Passa | Passa | Não há falha |
| `nscPinned` | Sim | Sim | Passa | Falha | SPKI não bate |

Essa tabela resume a distinção que eu quero provar.

<img src="/images/posts/02-por-que-instalar-a-ca-do-burp-nao-basta-para-interceptar-um-app-android/nsc-pinned-direct-vs-burp.png" alt="À esquerda, a flavor nscPinned passa na requisição direta para a API; à direita, falha quando a requisição passa pelo Burp" class="center" width="820">

*À esquerda, `nscPinned` passa na requisição direta para a API. À direita, a mesma flavor falha quando a requisição passa pelo Burp e o SPKI apresentado muda.*

## Pergunta 5: em que momento do fluxo cada falha acontece?

As duas falhas aparecem como problemas de TLS para a app, mas elas não significam a mesma coisa.

No cenário com proxy explícito, a app primeiro estabelece a conexão TCP (Transmission Control
Protocol) com o Burp e solicita um túnel com o método HTTP (Hypertext Transfer Protocol) `CONNECT`.
Depois, negocia TLS por esse túnel. Portanto, já existe HTTP de controle do proxy antes do TLS;
o que ainda não foi enviado é a requisição da aplicação, como `GET /profile`.

Para as flavors que usam `HttpsURLConnection` e os pins do NSC neste experimento, o fluxo é:
**TCP com o proxy → CONNECT → handshake TLS → validação da cadeia e dos pins do NSC → requisição HTTP da aplicação no túnel TLS.**

O ponto relevante para o diagnóstico é que a comunicação TCP com o proxy e o túnel podem funcionar enquanto a validação TLS rejeita a conexão. Isso impede o envio do `GET /profile`, sem significar que toda comunicação anterior falhou.

Em `systemOnly`, a falha acontece quando o Android tenta encontrar uma trust anchor: **TCP OK → TLS handshake começa → cadeia do Burp não chega a uma CA aceita → handshake aborta → GET /profile não é enviado pelo túnel TLS.**

Em `nscPinned`, a cadeia já é confiável, mas o Android aplica a verificação extra de pins: **TCP OK → TLS handshake começa → cadeia do Burp chega a uma CA aceita → `NetworkSecurityTrustManager.checkPins` rejeita o SPKI → handshake aborta → GET /profile não é enviado pelo túnel TLS.**

Essa diferença importa porque os sintomas visuais podem parecer parecidos: as duas aparecem como `FAILURE` com `SSLHandshakeException`. Mas o diagnóstico correto depende da causa raiz:

- neste caso, `CertPathValidatorException` com a mensagem `Trust anchor for certification path not found` → ausência de uma trust anchor aceita;
- `Pin verification failed` → falha real de pinning.

## Pergunta 6: uma build debug com `<pin-set>` também rejeita o Burp?

Depois de separar trust store de pinning, eu quero testar uma situação que costuma confundir análises Android: o que acontece quando uma build debug contém pins, mas também declara uma CA específica em `debug-overrides`?

A variante `debugOverrides` tem um `pin-set`:

```xml
<pin-set expiration="2035-01-01">
    <pin digest="SHA-256">Kyfpm+U4uD7FC1lNrw/7LHwerRcQ0EIMmaWCZd0dk8Q=</pin>
    <pin digest="SHA-256">jN72Sgrbs/bv6VCez+cHlzrPx3iW26rCyaTwmAJB0Tk=</pin>
</pin-set>
```

Mas também tem:

```xml
<debug-overrides>
    <trust-anchors>
        <certificates src="@raw/burp_ca" overridePins="true" />
    </trust-anchors>
</debug-overrides>
```

Eu executo:

```bash
adb shell am force-stop com.example.sslpinninglab.debugoverrides
adb shell am start \
  -n com.example.sslpinninglab.debugoverrides/com.example.sslpinninglab.MainActivity \
  --ez auto_request true
```

O resultado é `SUCCESS`, com `HTTP 200`.

<img src="/images/posts/02-por-que-instalar-a-ca-do-burp-nao-basta-para-interceptar-um-app-android/debug-overrides-burp-success.png" alt="Flavor debugOverrides funcionando via Burp apesar do pin-set" class="center" width="420">

*`debugOverrides` continua passando via Burp porque a CA de debug pode ignorar os pins.*

O logcat confirma: `debug-overrides request succeeded`.

Isso não significa que o pinning “aceita” o certificado do Burp.

O que acontece é:

1. a build é debuggable, isto é, uma build marcada para depuração;
2. a CA do Burp está declarada em `debug-overrides`;
3. a conexão usa essa CA de debug;
4. `overridePins="true"` instrui o Android a ignorar pins para essa anchor.

Essa dispensa de pins é a do NSC. Ela não garante que uma checagem adicional implementada em uma biblioteca ou em código próprio também será ignorada.

Vale uma ressalva: esse comportamento depende do atributo `android:debuggable` do build, não do nome da flavor em si. Qualquer variante buildada como debuggable que declare `debug-overrides` se comportaria da mesma forma, independentemente de qual flavor ela representa. No laboratório, `debugOverrides` só isola essa configuração para deixá-la fácil de observar.

Em outras palavras: *o pinning não vence; ele é deliberadamente pulado na configuração de debug.*

Esse detalhe é importante porque uma análise feita apenas em build debug pode levar a uma conclusão errada: *“a app aceita Burp, então ela não tem pinning”*.

Na verdade, a release pode continuar protegida, porque `debug-overrides` só participa da política quando a aplicação é debuggable.

## O que o experimento realmente me ensinou

A diferença entre as flavors deixa o diagnóstico observável: `systemOnly` rejeita a cadeia,
`proxyTrusted` a aceita, e `nscPinned` acrescenta uma restrição que rejeita os pins do proxy.
`debugOverrides` mostra uma exceção deliberada à aplicação dos pins do NSC numa build de debug.

Portanto, o mesmo resultado visual, `FAILURE`, pode esconder decisões diferentes. A causa raiz,
a configuração e a comparação com a baseline são o que me permitem distinguir essas decisões.

## Nota para quem for implementar pinning

O laboratório até aqui olha pela lente de quem analisa a app. Mas quem for configurar pinning em uma app real vai tomar algumas dessas mesmas decisões do outro lado:

- `debug-overrides` com `overridePins="true"` é útil para desenvolvimento, mas só é seguro se a flag `android:debuggable` realmente não vazar para o build de release;
- em `src="user"`, explicitar `overridePins="false"` documenta a intenção, mas esse já é o padrão fora de `debug-overrides`. Dentro de `debug-overrides`, o padrão é `true`. O bypass dos pins do NSC ocorre quando a cadeia usa uma fonte configurada com `overridePins="true"`;
- um `<pin-set>` com um único pin é um risco operacional: se a chave privada do servidor precisar rotacionar sem aviso, a app para de funcionar. O backup pin existe para cobrir exatamente esse cenário.

Esses pontos vão ganhar mais peso prático quando a série chegar a instrumentação e bypass de pinning, mas já valem como checklist para quem for revisar a configuração de uma app antes disso.

## Materiais e referências para reproduzir

Os experimentos dependem da API e das flavors deste laboratório. Os arquivos de apoio incluem
a [API HTTPS](#TODO-link-github-lab-api-readme), o [app Android](#TODO-link-github-lab-android-app-readme), a
[preparação do ambiente](#TODO-link-github-readme-environment-setup) e o
[roteiro completo de reprodução](#TODO-link-github-readme-full-walkthrough).

Para montar uma cópia própria, gerar novas chaves não basta: os pins dos XMLs precisam acompanhar
o SPKI das suas chaves, e a CA do Burp declarada em `debugOverrides` precisa ser a do seu proxy.
Os hashes e as capturas deste texto pertencem à execução documentada; não são valores que qualquer
instalação nova deva copiar sem conferir. Nas flavors deste artigo, eu altero os pins declarados
em `<pin-set>` conforme as chaves geradas e reconstruo os APKs antes de comparar os resultados.

Para conferir o comportamento da plataforma:

- [Android — Network Security Configuration](https://developer.android.com/privacy-and-security/security-config): anchors, pins e configurações de debug.
- [Android — atributo overridePins](https://developer.android.com/privacy-and-security/security-config#certificates): valores padrão e dispensa de pins por fonte de CAs.
- [Android — mudanças no Android 7.0](https://developer.android.com/about/versions/nougat/android-7.0-changes#tls-ssl): confiança padrão em CAs adicionadas pelo usuário.
- [Android Emulator — rede](https://developer.android.com/studio/run/emulator-networking): significado de `10.0.2.2` no emulador.

## Modelo mental final

Depois de percorrer o experimento, eu separo estas responsabilidades. É um modelo conceitual para este laboratório, sem impor uma ordem universal de execução entre hostname, cadeia e pins:

1. primeiro a app decide em quais CAs confiar;
2. depois o Android valida hostname, validade e cadeia;
3. se a cadeia chega a uma trust anchor aceita, a conexão ainda pode enfrentar uma checagem extra;
4. o pinning compara a chave pública apresentada com os SPKIs esperados;
5. uma falha antes dessa etapa não é pinning.

Ou, de forma ainda mais curta: *trust store responde “em quais CAs eu confio?”; pinning responde “essa chave pública é uma das esperadas?”*

Para atribuir esta falha ao pinning, eu preciso separar a rejeição da cadeia da rejeição dos pins.
A baseline é o que me permite isolar essa diferença neste experimento.

Isso não significa que uma app incapaz de aceitar a CA do proxy não possa conter pinning. Ela pode
ter os dois controles, com a primeira rejeição impedindo que a execução alcance a checagem dos
pins. Presença de pinning e causa da falha observada são perguntas diferentes.

Até aqui, eu provei apenas que o pinning bloqueia um MITM de rede enquanto a app continua executando sua própria política de confiança intacta. Eu ainda não provei o que acontece quando alguém controla o processo da app ou altera o APK.

Depois de provar que `nscPinned` falha por pinning real, a próxima pergunta deixa de ser “por que a conexão falha?” e passa a ser: *onde essa decisão vive dentro do APK (Android Package)?*

Essa é a investigação que eu quero abrir no próximo artigo.

## Referência rápida dos termos usados

| Termo | Significado |
| --- | --- |
| Android Emulator | Emulador usado para reproduzir o device Android |
| Burp Suite | Proxy de interceptação usado para observar tráfego HTTPS |
| MITM (Man-in-the-Middle) | Intermediação da conexão entre cliente e servidor |
| CA (Certificate Authority) | Autoridade certificadora que assina certificados |
| Trust store | Conjunto local de CAs confiáveis disponíveis para o cliente |
| Trust anchor | Certificado ou chave confiável que encerra uma validação concreta |
| System store | Conjunto de CAs confiadas pelo sistema Android |
| User store | Conjunto de CAs adicionadas pelo usuário no device |
| User-added CA | CA instalada manualmente pelo usuário |
| Flavor | Variante de build da mesma aplicação Android |
| `adb reverse` | Ponte que encaminha uma porta do emulador para o host |
| Network Security Config | XML Android que define trust anchors, cleartext e pins |
| APK (Android Package) | Pacote instalável de uma aplicação Android |
| `src="system"` | Fonte de CAs do sistema no NSC |
| `src="user"` | Fonte de CAs adicionadas pelo usuário no NSC |
| `<pin-set>` | Lista de pins SPKI aceitos para um domínio |
| SPKI (Subject Public Key Info) | Estrutura do certificado que descreve a chave pública |
| `debug-overrides` | Configuração especial de confiança para builds debuggable |
| `overridePins` | Define se uma anchor pode ignorar pins |
| `SSLHandshakeException` | Exceção que indica uma falha na negociação TLS |
| `CertPathValidatorException` | Exceção de validação da cadeia até uma trust anchor |
