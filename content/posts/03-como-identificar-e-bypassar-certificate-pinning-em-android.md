+++
title = "Como identificar e bypassar certificate pinning em Android"
translationKey = "android-certificate-pinning-identification-bypass"
date = "2026-11-11"
author = "Thiago Siqueira"
tags = ["android", "ssl-pinning", "mobile", "frida", "reverse-engineering"]
keywords = ["android", "certificate pinning", "frida", "apktool", "jadx", "okhttp"]
description = "Comparo duas implementações de certificate pinning em Android, Network Security Config e OkHttp CertificatePinner, para mostrar como análise estática e instrumentação dinâmica revelam pontos diferentes de bypass."
showFullContent = false
readingTime = true
hideComments = false
+++

No artigo anterior, eu provei duas coisas importantes:

1. uma app Android pode rejeitar a CA do Burp antes de qualquer pinning existir;
2. depois que a cadeia do proxy já é aceita, um `<pin-set>` ainda pode bloquear a conexão porque o SPKI apresentado pelo Burp não bate com o esperado pela app.

Isso resolve uma confusão comum: nem toda falha de TLS em Android é pinning.

Mas isso deixa uma pergunta nova:

*Se duas apps falham por pinning, isso significa que a verificação vive no mesmo lugar?*

Essa pergunta importa porque o sintoma visual pode ser quase idêntico: `FAILURE`, com uma exceção ao estabelecer a conexão segura.

Mesmo assim, a estratégia de análise e bypass pode mudar completamente.

No meu laboratório, eu construí duas flavors que rejeitam o Burp por pinning:

- `nscPinned`, que usa Network Security Config;
- `okhttpPinned`, que usa `CertificatePinner` da biblioteca OkHttp.

As duas chamam a mesma API, usam os mesmos pins e falham contra o mesmo proxy.

Uma declara a política em XML, para a plataforma aplicar. A outra configura a verificação no código do cliente HTTP.

É essa diferença que eu quero investigar neste artigo:

*Quando a conexão é rejeitada, onde a aplicação está aplicando essa política?*

Eu não introduzi OkHttp no artigo anterior porque ali a pergunta era outra: eu queria separar
trust store de pinning usando a política nativa do Android. Agora que essa distinção já está
provada, eu consigo adicionar uma segunda implementação sem misturar os conceitos.

## Antes de começar: o que continua igual no laboratório

Os comandos deste artigo partem da raiz do projeto, com os APKs já compilados e instalados, a API
local em execução e as CAs do laboratório e do Burp instaladas no user store do Android.

Uso um emulador Android API 36 com root, OkHttp **4.12.0**, Frida e `frida-server` **17.17.0**,
frida-tools **14.10.4**, apktool **3.0.3** e JADX **1.5.5**. O `frida-server` já está em execução
com privilégios de root; ele é o componente no device que permite ao cliente Frida instrumentar
o processo. Ter o proxy funcionando, por si só, não fornece esse acesso.

A preparação está descrita em [Ambiente do laboratório](#TODO-link-github-docs-00-environment). Antes dos comandos,
eu carrego os caminhos do SDK Android e do Java:

```bash
source scripts/lab-env.sh
```

Eu mantenho o mesmo ambiente do artigo anterior: **Android Emulator → Burp Suite no macOS → API HTTPS local**.

A API continua expondo `https://api.ssl-pinning.127.0.0.1.nip.io:8443/profile`.

O proxy continua configurado no Android:

```bash
adb shell settings put global http_proxy 10.0.2.2:8080
```

E a ponte do emulador até a API local continua ativa:

```bash
adb reverse tcp:8443 tcp:8443
```

Também continuo usando a mesma app com múltiplas flavors. Elas compartilham a tela e o endpoint. As duas variantes com pinning comparadas aqui, `nscPinned` e `okhttpPinned`, aplicam os mesmos pins:

- `sha256/Kyfpm+U4uD7FC1lNrw/7LHwerRcQ0EIMmaWCZd0dk8Q=`
- `sha256/jN72Sgrbs/bv6VCez+cHlzrPx3iW26rCyaTwmAJB0Tk=`

Isso é importante porque me permite comparar apenas o ponto de implementação.

A infraestrutura permanece a mesma; o ponto de implementação da política é o que muda.

## Antes de comparar: a baseline ainda passa

Antes de comparar duas falhas, eu revalido rapidamente a baseline do artigo anterior.

A flavor `proxyTrusted` aceita a CA do Burp e não tem pinning:

```bash
adb shell am force-stop com.example.sslpinninglab.proxytrusted
adb shell am start \
  -n com.example.sslpinninglab.proxytrusted/com.example.sslpinninglab.MainActivity \
  --ez auto_request true
```

O resultado continua sendo `SUCCESS`, com `HTTP 200`.

Isso é importante porque mantém a mesma conclusão do artigo anterior: *se outra flavor falhar agora, a causa já não é simplesmente "o Burp não funciona".*

## A primeira evidência: duas flavors falham contra o mesmo Burp

Eu começo repetindo a falha de `nscPinned`, que já apareceu no artigo anterior:

```bash
adb shell am force-stop com.example.sslpinninglab.nscpinned
adb shell am start \
  -n com.example.sslpinninglab.nscpinned/com.example.sslpinninglab.MainActivity \
  --ez auto_request true
```

O resultado é `FAILURE`, com `Exception: SSLHandshakeException` e causa raiz `CertificateException: Pin verification failed`.

Para acompanhar as exceções emitidas pela app, abro outro terminal antes da requisição:

```bash
adb logcat -v threadtime -s 'SslPinningLab:E' '*:S'
```

`SslPinningLab` é a tag usada pelo código do laboratório para registrar a exceção completa.
O filtro seleciona seus erros e silencia as demais tags. Ele pode incluir registros antigos;
para correlacionar a falha, observo o horário da nova requisição. Encerro a leitura com `Ctrl+C`.

O trecho relevante mostra `Caused by: java.security.cert.CertificateException: Pin verification failed`, em `NetworkSecurityTrustManager.checkPins(...)`.

Depois eu abro `okhttpPinned`:

```bash
adb shell am force-stop com.example.sslpinninglab.okhttppinned
adb shell am start \
  -n com.example.sslpinninglab.okhttppinned/com.example.sslpinninglab.MainActivity \
  --ez auto_request true
```

Na tela, a app também falha, com `FAILURE`, `Exception: SSLPeerUnverifiedException` e causa raiz `SSLPeerUnverifiedException: Certificate pinning failure!`.

O logcat confirma `javax.net.ssl.SSLPeerUnverifiedException: Certificate pinning failure!`.

À primeira vista, as duas variantes parecem contar a mesma história:

- o Burp apresenta uma chave pública diferente;
- a app esperava os SPKIs legítimos;
- a requisição HTTP da aplicação não chega a ser enviada pelo túnel TLS.

Essa distinção de camadas importa: o proxy recebe um `CONNECT` HTTP para abrir o túnel antes
mesmo da validação TLS. O que fica bloqueado aqui é o `GET /profile` da aplicação.

No OkHttp 4.12.0, o `CertificatePinner` é chamado depois do handshake do socket e da verificação
de hostname, antes de liberar a conexão para a requisição da aplicação. Já o NSC aplica os pins
na validação de confiança da plataforma. Portanto, falhar ao estabelecer uma conexão segura não
significa que as duas rejeições aconteçam na mesma etapa.

As exceções dão uma pista de que a implementação mudou:

| Variante | Exceção principal | Ponto visível no stack trace |
| --- | --- | --- |
| `nscPinned` | `SSLHandshakeException`, causada por `CertificateException` | `NetworkSecurityTrustManager.checkPins(...)` |
| `okhttpPinned` | `SSLPeerUnverifiedException` | `CertificatePinner.check$okhttp(...)` |

O problema lógico parece o mesmo.

O ponto de decisão ainda precisa ser confirmado.

Essa é a primeira ideia importante do artigo:

*O sintoma pode ser igual, mas a implementação muda a investigação.*

<img src="/images/posts/03-como-identificar-e-bypassar-certificate-pinning-em-android/pinning-failures-comparison.png" alt="Comparação entre as flavors nscPinned e okhttpPinned falhando contra o Burp por mecanismos diferentes" class="center" width="820">

*As duas flavors falham contra o mesmo Burp, mas a mensagem já sugere que a decisão está sendo tomada em pontos diferentes.*

Neste ponto, eu ainda não tratei a exceção como prova final.

Uma falha de handshake é uma pista. Ela me ajuda a decidir onde procurar, mas não substitui a
inspeção do APK.

| Evidência | O que sugere | O que ainda preciso confirmar |
| --- | --- | --- |
| Falha ao passar pelo Burp | A conexão foi rejeitada antes do HTTP normal da app | Qual validação tomou a decisão |
| `NetworkSecurityTrustManager.checkPins(...)` no stack trace | A política pode vir do Network Security Config | Se o XML realmente contém `<pin-set>` |
| `CertificatePinner.check$okhttp(...)` no stack trace | A biblioteca OkHttp pode estar aplicando pins | Se essa chamada participa do fluxo real do APK |

## Pergunta 1: onde eu procuro primeiro dentro do APK?

Quando eu tenho um APK e quero entender onde o pinning vive, eu não começo tentando rodar um script universal de bypass.

Eu começo procurando evidências estáticas.

Para isso, uso duas ferramentas com papéis diferentes:

| Ferramenta | O que eu quero dela |
| --- | --- |
| `apktool` | Manifest, resources XML e smali decodificados do APK |
| `jadx` | Uma visão Java/Kotlin aproximada do bytecode |

Eu organizo os arquivos gerados por experimento, dentro de `/tmp/ssl-pinning-lab`:

| Diretório | Conteúdo |
| --- | --- |
| `/tmp/ssl-pinning-lab/static-analysis` | Recursos decodificados pelo apktool e código exportado pelo JADX |
| `/tmp/ssl-pinning-lab/nsc-patch` | Cópia editável do APK NSC e APKs reconstruídos |

Esses diretórios são áreas de trabalho locais, separadas dos fontes em `lab/`. Os nomes
identificam o experimento e podem ser usados tanto neste artigo quanto na reprodução do
laboratório. Como ficam em `/tmp`, podem ser removidos pelo sistema; não guardo ali a única
cópia de uma evidência que eu queira preservar.

Eu decodifico os três APKs principais do laboratório:

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

O comando `apktool d` decodifica o APK.

Ele não recupera o projeto original.

O que ele me entrega é uma representação reconstruída dos recursos e do bytecode, suficiente para
seguir referências como manifesto, XML e smali.

Os parâmetros mais importantes são:

| Trecho | Função |
| --- | --- |
| `-f` | Sobrescreve uma extração anterior |
| `-p /tmp/.../framework` | Mantém os frameworks do apktool em uma pasta isolada |
| `-o /tmp/...` | Define o diretório onde os recursos decodificados serão gravados |

Depois eu abro a variante OkHttp no jadx:

```bash
jadx -d /tmp/ssl-pinning-lab/static-analysis/jadx/okhttpPinned \
  lab/android-app/app/build/outputs/apk/okhttpPinned/debug/app-okhttpPinned-debug.apk
```

Para localizar a referência ao NSC nos manifestos decodificados, uso:

```bash
grep -n 'networkSecurityConfig' /tmp/ssl-pinning-lab/static-analysis/apktool/*/AndroidManifest.xml
```

A busca aponta para o atributo abaixo no `AndroidManifest.xml` das três variantes:

```xml
<application
    android:networkSecurityConfig="@xml/network_security_config"
    ... />
```

Isso ainda não prova pinning.

Só prova que existe um Network Security Config ativo que vale ser lido.

Encontrar um arquivo chamado `network_security_config.xml` por si só não bastaria. O que importa
é confirmar que o manifesto realmente aponta para ele.

## Pergunta 2: o que o XML revela sobre `nscPinned`?

Eu sigo a referência do manifesto até o recurso decodificado:

```bash
cat /tmp/ssl-pinning-lab/static-analysis/apktool/proxyTrusted/res/xml/network_security_config.xml
cat /tmp/ssl-pinning-lab/static-analysis/apktool/nscPinned/res/xml/network_security_config.xml
```

Na baseline `proxyTrusted`, o XML extraído pelo apktool mostra:

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

Não há `<pin-set>`.

Essa variante aceita a CA do Burp, mas não exige um SPKI específico.

Em `nscPinned`, o XML é quase igual, mas ganha um `domain-config` com pins:

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

Nesse caso, o APK me conta quase tudo sem eu precisar abrir nenhuma classe Java:

1. a app aceita CAs do sistema e do user store;
2. o domínio da API está explicitamente coberto por pins;
3. os hashes aparecem diretamente no XML;
4. o ponto de bloqueio é uma política declarativa do Android.

Isso também explica por que o stack trace apontava para `NetworkSecurityTrustManager.checkPins(...)`.

Quem aplica a decisão não é uma classe escrita pela app.

É a própria plataforma Android, lendo a política do XML.

Nesse ponto, o bypass mais natural já começa a aparecer: *se o pin vive no XML, remover o pin do XML é uma hipótese razoável.*

## Pergunta 3: e se o XML não tiver `<pin-set>`?

Agora eu abro o XML de `okhttpPinned`:

```bash
cat /tmp/ssl-pinning-lab/static-analysis/apktool/okhttpPinned/res/xml/network_security_config.xml
```

Ele é igual ao da baseline:

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

Não há `<pin-set>`.

Se eu parasse aqui, eu concluiria errado: *essa app não tem pinning.*

Mas eu já vi a app falhar com `SSLPeerUnverifiedException: Certificate pinning failure!`.

Isso me indica que a decisão vive em outro lugar.

Nos arquivos exportados pelo JADX, procuro tanto a construção dos pins quanto sua associação
ao cliente:

```bash
grep -rnE 'CertificatePinner|certificatePinner' /tmp/ssl-pinning-lab/static-analysis/jadx/okhttpPinned/sources
```

A busca aponta para `MainActivity`. Encontrar a classe da biblioteca empacotada no APK não bastaria:
eu preciso seguir seu uso no cliente que executa a requisição.

Para inspecionar esse trecho na interface gráfica, também posso abrir o APK diretamente:

```bash
jadx-gui lab/android-app/app/build/outputs/apk/okhttpPinned/debug/app-okhttpPinned-debug.apk
```

Na árvore de código, abro `com.example.sslpinninglab.MainActivity` e localizo
`requestWithOkHttpPinning`. O trecho relevante, formatado para leitura, é:

```java
CertificatePinner certificatePinner = new CertificatePinner.Builder()
        .add(BuildConfig.API_HOST, BuildConfig.LEAF_PIN, BuildConfig.BACKUP_PIN)
        .build();

OkHttpClient client = new OkHttpClient.Builder()
        .certificatePinner(certificatePinner)
        .build();
```

Agora a diferença fica clara:

| Caso | Onde o pin vive | Quem aplica a validação |
| --- | --- | --- |
| `nscPinned` | `res/xml/network_security_config.xml` | Android / Network Security Config |
| `okhttpPinned` | Código Java/Kotlin | `okhttp3.CertificatePinner` |

O `<pin-set>` do Network Security Config declara os pins em XML; o NSC também configura outras
políticas de rede e pode existir sem pinning. Já o `CertificatePinner` configura os pins
programaticamente.

Nos dois casos, a app está pedindo a mesma coisa: *aceite apenas uma cadeia que contenha um SPKI esperado.*

Mas a forma como essa regra entra no APK é diferente.

E essa diferença muda o bypass.

<img src="/images/posts/03-como-identificar-e-bypassar-certificate-pinning-em-android/jadx-okhttp-certificate-pinner.png" alt="JADX mostrando o CertificatePinner sendo construído dentro de MainActivity" class="center" width="820">

*No JADX, a política de pinning aparece no próprio código da app: `CertificatePinner.Builder().add(...)` monta os pins e o cliente OkHttp recebe essa configuração.*

## Pergunta 4: por que eu não começo por um script universal?

Depois de encontrar `CertificatePinner`, seria tentador baixar um script pronto e torcer para ele funcionar.

Às vezes funciona.

Mas isso não ensina onde a decisão acontece.

No meu laboratório, eu quero provar o menor ponto de alteração necessário. Então eu parto do stack trace.

A falha da flavor OkHttp mostra `SSLPeerUnverifiedException: Certificate pinning failure!`, em `okhttp3.CertificatePinner.check$okhttp(...)`.

Esse detalhe importa, mas não porque `check$okhttp` seja uma API universal.

Esse é o método que aparece nesta versão do OkHttp e neste caminho de execução do laboratório.
Em outra versão, em outra build ou em uma app ofuscada, o nome ou a assinatura podem mudar.

`CertificatePinner` possui mais de um método relacionado à checagem:

- `check(java.lang.String, java.util.List)`
- `check(java.lang.String, java.security.cert.Certificate[])`
- `check$okhttp(java.lang.String, kotlin.jvm.functions.Function0)`

O stack trace já indica qual método participou da falha. Agora eu quero confirmar se ele está
disponível no processo com a assinatura que pretendo interceptar. Enumerar métodos confirma sua
existência; não demonstra, por si só, que foram executados.

Por isso eu escrevo um script de inspeção:

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

Esse script apenas inspeciona os métodos. Ele responde à pergunta: *quais métodos essa classe realmente expõe no processo que está rodando?*

Eu executo:

```bash
frida -U -n 'SSL Lab - OkHttp Pinned' -l lab/frida/inspect-okhttp.js -q
```

Os parâmetros significam:

| Trecho | Função |
| --- | --- |
| `-U` | Conecta ao device USB, no caso o emulador |
| `-n 'SSL Lab - OkHttp Pinned'` | Anexa ao processo já aberto pelo nome |
| `-l lab/frida/inspect-okhttp.js` | Carrega o script JavaScript |
| `-q` | Executa em modo quiet, deixando a saída mais limpa |

A saída mostra `[*] okhttp3.CertificatePinner declared methods:`, seguido de:

- `public final void okhttp3.CertificatePinner.check(java.lang.String,java.util.List)`
- `public final void okhttp3.CertificatePinner.check(java.lang.String,java.security.cert.Certificate[])`
- `public final void okhttp3.CertificatePinner.check$okhttp(java.lang.String,kotlin.jvm.functions.Function0)`

Agora eu tenho duas evidências apontando para o mesmo lugar:

1. o stack trace da falha passa por `check$okhttp`;
2. o runtime confirma que esse método existe com a assinatura esperada.

Com a falha localizada e a assinatura confirmada, eu consigo preparar o hook direcionado.

O script também fica no laboratório em `lab/frida/inspect-okhttp.js`.

Assim, a saída da captura de tela não vira a única evidência reproduzível da investigação.

## Pergunta 5: qual é o menor hook que prova a hipótese?

Antes do hook, uma pré-condição importa: neste flavor, a CA do Burp já é aceita pela política de
confiança da app.

Se isso não fosse verdade, remover apenas o pinning não bastaria.

O hook que eu vou testar não torna uma cadeia inválida em válida. Ele só impede a verificação
adicional de SPKI feita pelo `CertificatePinner`.

Se `check$okhttp` é o ponto em que a biblioteca rejeita o SPKI apresentado, eu não preciso reescrever toda a pilha TLS.

Eu só preciso neutralizar essa decisão.

O hook fica:

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

Para entender por que esse hook funciona, eu preciso olhar para o contrato do método original.
`check$okhttp` não devolve um booleano dizendo se os pins foram aceitos. Ele retorna normalmente
quando a checagem permite continuar e lança `SSLPeerUnverifiedException` quando rejeita os pins.

O segundo parâmetro, `Function0`, é uma função Kotlin sem argumentos que fornece a lista de
certificados para a verificação. Ao substituir o método inteiro, meu hook não chama essa função
nem executa a comparação original.

O `return;` encerra um método cujo retorno Java é `void`. Para o chamador, a invocação termina
normalmente, sem a exceção que interromperia a conexão. É isso que neutraliza a restrição neste
fluxo; não há uma nova validação dos certificados dentro do hook.

O script faz quatro coisas:

1. aguarda o runtime Java e executa a função de instrumentação com `Java.perform`;
2. obtém a classe `okhttp3.CertificatePinner`;
3. seleciona exatamente o overload observado no runtime;
4. substitui a implementação por um retorno vazio.

Eu anexo o hook:

```bash
frida -U -n 'SSL Lab - OkHttp Pinned' -l lab/frida/targeted-okhttp-hook.js
```

Esse segundo script também está versionado no laboratório em `lab/frida/targeted-okhttp-hook.js`.

Mantenho essa sessão aberta e aguardo a confirmação de que o hook foi instalado: `[*] Targeted OkHttp CertificatePinner hook installed`.

Só depois toco novamente em **REQUEST PROFILE**, na tela da app já instrumentada. Isso dispara
uma nova chamada sem encerrar o processo ao qual o Frida está anexado. Não uso `force-stop`
neste momento, porque isso encerraria também o processo instrumentado.

O Frida mostra novamente `[*] Targeted OkHttp CertificatePinner hook installed` e, na sequência, `[+] Bypassing CertificatePinner.check$okhttp for api.ssl-pinning.127.0.0.1.nip.io`.

E a app muda de estado para `SUCCESS`, com `HTTP 200`.

O payload volta a aparecer na tela.

<img src="/images/posts/03-como-identificar-e-bypassar-certificate-pinning-em-android/okhttp-after-hook.png" alt="Flavor okhttpPinned funcionando via Burp depois do hook no CertificatePinner" class="center" width="420">

*Depois do hook, a mesma flavor volta a responder com `HTTP 200` e o payload da API.*

Eu confiro também o histórico HTTP do Burp. A entrada mostra `GET /profile`,
`User-Agent: okhttp/4.12.0` e a resposta `HTTP/1.1 200 OK`. O campo `time` do JSON,
`2026-11-11T22:57:50.385Z`, coincide com o payload exibido pela app após o hook.
Essa correlação liga a execução instrumentada à requisição e à resposta legíveis no proxy.

<img src="/images/posts/03-como-identificar-e-bypassar-certificate-pinning-em-android/burp-okhttp-after-hook.png" alt="Histórico do Burp mostrando GET /profile com User-Agent okhttp/4.12.0 e resposta HTTP 200 cujo horário corresponde ao sucesso após o hook" class="center" width="1000">

*A entrada 2856 do Burp registra a chamada OkHttp após o hook. O horário no JSON é o mesmo observado na app.*

Eu removi a checagem adicional de pinning feita pela biblioteca. O handshake continua acontecendo
e a CA do Burp continua sendo aceita pela política de confiança da app; o que mudou foi a decisão
local que comparava os SPKIs.

Isso confirma o papel do `CertificatePinner` neste fluxo específico. Falhas em outras verificações
TLS continuam exigindo investigação própria.

### O que acontece quando eu reinicio sem o hook?

Para conferir se a alteração ficou restrita ao processo, encerro a sessão do Frida com `Ctrl+C`
e paro a app. Depois abro novamente o mesmo APK, sem anexar o script:

```bash
adb shell am force-stop com.example.sslpinninglab.okhttppinned
adb shell am start \
  -n com.example.sslpinninglab.okhttppinned/com.example.sslpinninglab.MainActivity \
  --ez auto_request true
```

O resultado volta a ser `FAILURE`, com `SSLPeerUnverifiedException: Certificate pinning failure!`.

O proxy e o APK continuam os mesmos. Ao encerrar o processo instrumentado, a implementação
original volta a ser usada na próxima execução. A sequência observada fica:

| Estado | Resultado |
| --- | --- |
| `proxyTrusted`, sem pinning, via Burp | `SUCCESS`, HTTP 200 |
| `okhttpPinned`, antes do hook | `FAILURE`, `Certificate pinning failure!` |
| `okhttpPinned`, com o hook executado | `SUCCESS`, HTTP 200 |
| Mesmo APK reiniciado sem o hook | `FAILURE`, `Certificate pinning failure!` |

<img src="/images/posts/03-como-identificar-e-bypassar-certificate-pinning-em-android/okhttp-restarted-without-hook.png" alt="Flavor okhttpPinned voltando a rejeitar os pins do Burp depois do reinício sem instrumentação" class="center" width="420">

*O reinício sem hook restaura a rejeição. A instrumentação anterior não alterou o APK instalado.*

As telas, os XMLs e a linha do tempo dessa validação estão nas
[evidências complementares](#TODO-link-github-evidence-article-03-review-readme). O log do Frida registra a
execução do hook para o hostname da API; a resposta exibida pela app contém o horário da nova
requisição. A captura do histórico do Burp permite correlacionar esse mesmo payload com o HTTP observado no proxy.

## Pergunta 6: o bypass de `nscPinned` segue a mesma lógica?

A lógica continua sendo neutralizar a restrição adicional, mas eu escolho outro ponto de
intervenção.

Em `nscPinned`, a política é declarada no XML e executada por código da plataforma Android, como
mostra `NetworkSecurityTrustManager.checkPins`. Para este experimento, o bypass mais didático é
remover o `<pin-set>` antes da execução.

Isso não significa que NSC só possa ser contornado por patch ou que OkHttp só possa ser
contornado por hook. Também seria possível investigar a validação da plataforma em runtime ou
alterar no APK o código que configura o OkHttp. Aqui eu escolho duas intervenções para comparar
uma alteração em memória com uma alteração no artefato.

Eu altero o artefato, reconstruo o APK e só depois executo a app novamente.

Eu preparo uma área separada da análise estática e decodifico o APK:

```bash
mkdir -p /tmp/ssl-pinning-lab/nsc-patch/framework

apktool d -f \
  -p /tmp/ssl-pinning-lab/nsc-patch/framework \
  lab/android-app/app/build/outputs/apk/nscPinned/debug/app-nscPinned-debug.apk \
  -o /tmp/ssl-pinning-lab/nsc-patch/nscPinned-decoded
```

O arquivo relevante aparece em `/tmp/ssl-pinning-lab/nsc-patch/nscPinned-decoded/res/xml/network_security_config.xml`.

O trecho original contém:

```xml
<pin-set expiration="2035-01-01">
    <pin digest="SHA-256">Kyfpm+U4uD7FC1lNrw/7LHwerRcQ0EIMmaWCZd0dk8Q=</pin>
    <pin digest="SHA-256">jN72Sgrbs/bv6VCez+cHlzrPx3iW26rCyaTwmAJB0Tk=</pin>
</pin-set>
```

Eu removo apenas esse bloco e mantenho o restante da política igual. Para conferir o alcance
da edição, comparo o XML original do projeto com o recurso que acabei de modificar:

```bash
diff -u \
  lab/android-app/app/src/nscPinned/res/xml/network_security_config.xml \
  /tmp/ssl-pinning-lab/nsc-patch/nscPinned-decoded/res/xml/network_security_config.xml
```

A comparação pode incluir diferenças de formatação do apktool; a alteração de política que eu
quero ver é a remoção do `<pin-set>`, preservando os domínios e as trust anchors.

No APK de laboratório que estou reconstruindo, isso testa o efeito de retirar a política do
artefato modificado. Não altera o APK original que já existia antes da edição.

Depois reconstruo o APK:

```bash
apktool b -p /tmp/ssl-pinning-lab/nsc-patch/framework \
  /tmp/ssl-pinning-lab/nsc-patch/nscPinned-decoded \
  -o /tmp/ssl-pinning-lab/nsc-patch/app-nscPinned-patched-unsigned.apk
```

O APK reconstruído ainda precisa ser alinhado e assinado. Antes de executar os comandos,
eu separo o papel de cada etapa:

| Comando | O que faz neste experimento |
| --- | --- |
| `zipalign` | Ajusta o alinhamento dos dados não comprimidos dentro do ZIP que forma o APK, permitindo acesso eficiente a esses arquivos. Não assina o pacote. |
| `apksigner sign` | Assina o APK reconstruído com a chave privada da debug keystore e inclui o certificado do signatário. A assinatura permite ao Android verificar a integridade do pacote e identificar a chave que o assinou. |
| `apksigner verify --print-certs` | Verifica a assinatura do APK e mostra informações sobre o certificado de assinatura. Não testa o funcionamento da app nem seu pinning TLS. |

Eu executo as etapas nessa ordem: alinhamento, assinatura e verificação.

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

A alteração do recurso protegido pela assinatura invalida a assinatura anterior; por isso o APK
reconstruído precisa ser assinado novamente. Eu alinho antes de assinar, porque o alinhamento
modifica o pacote e, se fosse feito depois, invalidaria a assinatura produzida pelo `apksigner`.
A verificação final confere o resultado antes da instalação. Essa ordem está descrita nas
referências oficiais de [zipalign](https://developer.android.com/tools/zipalign) e
[apksigner](https://developer.android.com/tools/apksigner).

O certificado de assinatura do APK tem uma função diferente do certificado TLS da API: o primeiro
identifica a chave usada para assinar o pacote; o segundo associa a chave pública do servidor ao
domínio na autenticação TLS. Assinar o APK alterado não muda o certificado da API nem os pins que
a app espera encontrar.

No laboratório, a build original e o APK patched usam a mesma debug keystore, `~/.android/debug.keystore`.

Por isso `adb install -r` consegue substituir a app instalada sem exigir desinstalação.

Em uma app real, um APK modificado normalmente não conseguiria substituir a instalação oficial,
porque o atacante não teria a chave privada de assinatura de produção. O bypass continua sendo
relevante quando a pré-condição inclui controle da instalação, do device ou do ambiente de teste.

Depois eu instalo o APK alterado:

```bash
adb install -r /tmp/ssl-pinning-lab/nsc-patch/app-nscPinned-patched.apk
adb shell am force-stop com.example.sslpinninglab.nscpinned
adb shell am start \
  -n com.example.sslpinninglab.nscpinned/com.example.sslpinninglab.MainActivity \
  --ez auto_request true
```

O resultado muda para `SUCCESS`, com `HTTP 200`.

De novo, eu não tornei o Burp “mais confiável”.

Eu apenas removi a decisão adicional que rejeitava o SPKI apresentado pelo proxy.

<img src="/images/posts/03-como-identificar-e-bypassar-certificate-pinning-em-android/nsc-patch-before-after.png" alt="Comparação entre nscPinned antes e depois da remoção do pin-set no APK" class="center" width="820">

*Antes do patch, a política declarativa ainda rejeita o SPKI do Burp. Depois de remover o `<pin-set>` e reinstalar o APK, a mesma requisição passa.*

## O que a comparação entre os dois bypasses revela

A diferença central do laboratório fica:

| Variante | Onde o pinning vive | Evidência estática | Evidência dinâmica | Bypass demonstrado |
| --- | --- | --- | --- | --- |
| `nscPinned` | XML | `<pin-set>` em `network_security_config.xml` | `NetworkSecurityTrustManager.checkPins` | Remover `<pin-set>` e rebuildar o APK |
| `okhttpPinned` | Código Java/Kotlin | `CertificatePinner.Builder().add(...)` | `CertificatePinner.check$okhttp` | Hook Frida no overload real |

Essa tabela responde a pergunta principal do artigo: *duas apps podem falhar pelo mesmo motivo lógico e oferecer pontos diferentes de intervenção.*

As técnicas usadas aqui dependem da implementação e do acesso disponível. Mudanças de versão,
ofuscação, classes customizadas ou validações nativas podem exigir outros pontos de intervenção.
A associação entre XML e patch, ou entre OkHttp e hook, descreve estes experimentos; não é uma
regra exclusiva para analisar qualquer APK.

O que continua válido em todos esses casos é a metodologia:

1. identificar a implementação;
2. localizar o ponto exato de decisão;
3. escolher o menor bypass que prova a hipótese;
4. validar se o comportamento mudou.

## Fontes para conferir os mecanismos

- [Android — Network Security Configuration](https://developer.android.com/privacy-and-security/security-config): políticas de confiança e pinning declaradas no XML.
- [OkHttp 4.12.0 — CertificatePinner](https://github.com/square/okhttp/blob/parent-4.12.0/okhttp/src/main/kotlin/okhttp3/CertificatePinner.kt): comparação dos pins, retorno normal e exceção de rejeição.
- [OkHttp 4.12.0 — RealConnection](https://github.com/square/okhttp/blob/parent-4.12.0/okhttp/src/main/kotlin/okhttp3/internal/connection/RealConnection.kt): ordem entre handshake, verificação de hostname e pinning.
- [Frida — JavaScript API](https://frida.re/docs/javascript-api/): uso de `Java.perform`, `Java.use` e substituição de métodos.

## O que esses bypasses provam

Ao final do experimento, eu consigo provar:

1. `nscPinned` e `okhttpPinned` bloqueiam o mesmo MITM por mecanismos diferentes;
2. análise estática é o que revela onde a regra de pinning vive;
3. o stack trace ajuda a formular a hipótese sobre o ponto dinâmico real da falha;
4. um hook Frida direcionado pode neutralizar a decisão de uma biblioteca em runtime;
5. um patch de APK pode remover uma política declarativa antes da execução;
6. pinning aumenta o custo do MITM, mas não é uma barreira absoluta contra alguém que controla o processo ou o APK.

## Modelo mental final

Depois dessa etapa do laboratório, a forma mais útil de pensar em bypass é:

- *O erro visual me diz que há uma falha.*
- *O stack trace me ajuda a localizar onde procurar.*
- *O APK confirma como ela foi implementada.*
- *O bypass precisa atacar exatamente esse ponto.*

Ou, de forma ainda mais curta:

- *A implementação revela onde intervir.*
- *O acesso disponível orienta como intervir.*
- *A mudança observada testa a hipótese.*

O valor não está em decorar um script.

O valor está em observar a falha, localizar a política, confirmar como ela participa do fluxo e só
então escolher a técnica de teste.

Depois de provar que o bypass depende da implementação, a próxima pergunta deixa de ser "como eu removo o pinning?" e passa a ser: *o que acontece quando a chave legítima do servidor precisa mudar?*

Essa é a investigação que eu quero abrir no próximo artigo.

## Referência rápida dos termos usados

| Termo | Significado |
| --- | --- |
| APK (Android Package) | Pacote instalável de uma aplicação Android |
| apktool | Ferramenta para decodificar e reconstruir APKs |
| jadx | Decompilador que reconstrói uma visão Java/Kotlin do bytecode |
| Network Security Config | XML Android que define trust anchors, cleartext e pins |
| `<pin-set>` | Lista de pins SPKI aceitos para um domínio |
| `CertificatePinner` | Classe do OkHttp usada para aplicar pinning programático |
| Frida | Toolkit de instrumentação dinâmica |
| Hook | Substituição de um comportamento em runtime |
| Overload | Variante de um método com assinatura diferente |
| `check$okhttp` | Método interno do OkHttp observado no runtime do laboratório |
| `SSLPeerUnverifiedException` | Exceção de verificação da identidade do peer; neste fluxo, mensagem e stack trace apontam para rejeição dos pins |
| `NetworkSecurityTrustManager.checkPins` | Método da plataforma Android que aplica pins do NSC |
| `zipalign` | Ferramenta que alinha o APK antes da assinatura |
| `apksigner` | Ferramenta que assina e valida APKs |
| Debug keystore | Chave de assinatura usada em builds Android de debug |
