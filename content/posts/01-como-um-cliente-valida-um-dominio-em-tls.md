+++
title = "Como um cliente valida a autenticidade de um domínio durante o handshake TLS"
translationKey = "tls-handshake-domain-validation"
date = "2026-10-10"
author = "Thiago Siqueira"
tags = ["tls", "pki", "criptografia", "openssl", "redes"]
keywords = ["tls", "pki", "certificados", "openssl", "handshake"]
description = "Uma PoC real com mercadopago.com.br para entender, passo a passo, como o OpenSSL valida a cadeia de certificados de um domínio e o que significa confiar em uma trust anchor local."
showFullContent = false
readingTime = true
hideComments = false
+++

Quando abrimos `https://mercadopago.com.br` no navegador, ou quando um aplicativo faz uma requisição HTTPS (Hypertext Transfer Protocol Secure) para uma API (Application Programming Interface), existe uma pergunta que precisa ser respondida antes de qualquer dado sensível ser trocado.

Antes de a conexão ser aceita, cliente e servidor passam pelo handshake, a negociação inicial do TLS em que parâmetros criptográficos são combinados e a identidade do servidor é validada. É nesse momento que surge a pergunta:

```text
Como o cliente sabe que está falando com o servidor legítimo daquele domínio?
```

É comum resumir TLS (Transport Layer Security) como “o protocolo que criptografa a conexão”. Essa definição não está errada, mas está incompleta. Criptografia, sozinha, não prova identidade. Um atacante também pode estabelecer uma conexão criptografada com o cliente. O que importa é saber se a chave pública apresentada do outro lado pertence, de fato, ao domínio que o cliente pretendia acessar.

Esse mecanismo depende de uma infraestrutura de chaves públicas, ou PKI (Public Key Infrastructure). Dentro dela, autoridades certificadoras, ou CAs (Certificate Authorities), assinam certificados para ligar nomes de domínio a chaves públicas.

Este artigo parte de uma PoC (Proof of Concept) real que eu realizei com `mercadopago.com.br`. A ideia não é decorar a cadeia específica observada, porque certificados públicos mudam com o tempo. A ideia é acompanhar a investigação passo a passo e responder, em cada etapa, o que o cliente está realmente verificando.

O fio condutor será sempre o mesmo:

```text
O que o servidor enviou?
O que o cliente verificou?
Em que momento ele decidiu confiar?
```

## Antes de começar: o que eu quis reproduzir

Meu objetivo é capturar a cadeia apresentada por um servidor real e entender como o OpenSSL consegue validá-la localmente.

O experimento usa:

- `openssl s_client` para abrir uma conexão TLS;
- `openssl x509` para inspecionar certificados;
- `openssl verify` para reproduzir a validação;
- o trust store local do OpenSSL instalado via Homebrew, isto é, o repositório local de CAs que esse OpenSSL já aceita como confiáveis.

Eu crio um diretório de trabalho:

```bash
mkdir -p ~/tmp/cert-lab-mercadopago
cd ~/tmp/cert-lab-mercadopago

openssl version -a
```

Na máquina usada no exercício, a saída relevante foi:

```text
OPENSSLDIR: "/opt/homebrew/etc/openssl@3"
```

Esse detalhe parece pequeno, mas vai ser importante mais tarde. O OpenSSL do Homebrew usa seu próprio bundle de CAs. Ele não necessariamente consulta o mesmo trust store usado por APIs nativas do macOS, pelo Chrome, pelo Java ou por uma aplicação Android.

## Pergunta 1: o que o servidor realmente envia?

Eu começo abrindo uma conexão TLS com o servidor e pedindo para o OpenSSL mostrar todos os certificados apresentados:

```bash
openssl s_client \
  -connect mercadopago.com.br:443 \
  -servername mercadopago.com.br \
  -showcerts \
  </dev/null 2>&1 | tee s_client.txt
```

Cada parte do comando tem um papel:

| Trecho | Função |
|---|---|
| `-connect mercadopago.com.br:443` | Define host e porta |
| `-servername mercadopago.com.br` | Envia SNI (Server Name Indication) para o servidor escolher o certificado correto |
| `-showcerts` | Mostra os certificados enviados pelo servidor |
| `</dev/null` | Evita que o comando fique esperando entrada interativa |
| `tee s_client.txt` | Salva a saída para análise posterior |

O parâmetro `-servername` merece atenção. SNI (Server Name Indication) é a extensão do TLS que envia o hostname desejado logo no início da conexão. Como vários domínios podem compartilhar o mesmo endereço IP (Internet Protocol), o servidor usa essa informação para decidir qual certificado deve apresentar.

O arquivo `s_client.txt` contém os certificados em formato PEM (Privacy-Enhanced Mail), uma representação textual dos dados binários do certificado. Para separar cada certificado em um arquivo próprio, eu uso:

```bash
awk '
/-----BEGIN CERTIFICATE-----/ {n++; file=sprintf("cert-%02d.pem", n)}
file {print > file}
/-----END CERTIFICATE-----/ {close(file); file=""}
' s_client.txt
```

O resultado foi:

```text
cert-01.pem
cert-02.pem
cert-03.pem
```

Até aqui, a única coisa que eu sei é:

```text
o servidor enviou três certificados
```

Ainda não sei qual é o leaf, qual é o intermediate e se algum deles é uma root.

Antes de continuar, vale fixar a nomenclatura. Leaf é o certificado do servidor final, aquele mais próximo do domínio acessado. Intermediate é uma CA intermediária, usada para assinar certificados leaf. Root é a CA no topo da hierarquia, normalmente usada como raiz da confiança.

## Pergunta 2: como descobrir o papel de cada certificado?

Eu inspeciono `subject`, `issuer`, `serial` e validade de cada um:

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

A saída observada foi:

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

Daqui eu já consigo montar uma primeira leitura:

| Arquivo | Papel provável | Por quê |
|---|---|---|
| `cert-01.pem` | Leaf | O `subject` é o domínio final |
| `cert-02.pem` | Intermediate | O `subject` é a CA que assinou o leaf |
| `cert-03.pem` | Certificado da Amazon Root CA 1 | O `subject` é `Amazon Root CA 1` |

Nos campos acima, `subject` identifica a entidade representada pelo certificado e `issuer` identifica quem o emitiu. O atributo `CN` (Common Name) aparece dentro desses nomes e historicamente foi usado para indicar o nome principal associado ao certificado.

Mas repare na palavra “provável”. Ver o `subject` e o `issuer` ajuda a entender a cadeia, mas não é a prova criptográfica em si.

O campo `issuer` é apenas uma declaração dentro do certificado. O que realmente prova a relação entre dois certificados é a assinatura digital.

Em outras palavras:

```text
cert-01 diz que foi emitido por Amazon RSA 2048 M04
mas o cliente ainda precisa verificar se a assinatura realmente foi feita pela chave privada daquela CA
```

## Pergunta 3: como eu sei que o leaf é válido para o domínio acessado?

O certificado leaf não é aceito apenas porque está no início da cadeia. Ele precisa representar o hostname correto.

Eu verifico o SAN (Subject Alternative Name) do leaf:

```bash
openssl x509 -in cert-01.pem -noout -ext subjectAltName
```

Saída:

```text
X509v3 Subject Alternative Name:
    DNS:www.mercadopago.com.br, DNS:mercadopago.com.br
```

Isso responde uma pergunta importante:

```text
O certificado leaf cobre o hostname que eu acessei?
```

Sim.

SAN é a extensão do certificado que lista os nomes de domínio para os quais ele é válido. O prefixo `DNS:` na saída indica que aqueles valores são nomes de domínio aceitos pelo certificado. Como eu acessei `mercadopago.com.br`, o cliente espera encontrar esse hostname no SAN do leaf. Se o certificado tivesse sido emitido para outro domínio, a conexão deveria ser rejeitada, mesmo que toda a cadeia de assinaturas fosse válida.

Historicamente, o campo `CN` (Common Name) também era usado para indicar o domínio principal. Na validação moderna, porém, o cliente deve comparar o hostname com o SAN, não apenas com o CN.

Também existem certificados com wildcard, como `*.mercadopago.com.br`. Nesse caso, o wildcard cobre um único nível à esquerda, como `api.mercadopago.com.br`, mas não cobre algo como `a.b.mercadopago.com.br`. As regras de matching de hostname são mais estritas do que simplesmente procurar um sufixo parecido.

Esse é um ponto importante: a autenticação TLS não é apenas “a cadeia foi assinada por uma CA confiável”. Ela também é:

```text
essa cadeia confiável representa o hostname que eu pedi?
```

## Pergunta 4: como nasce uma cadeia dessas?

Antes de continuar com a PoC, vale entender como esses certificados foram criados.

Uma autoridade certificadora, ou CA, começa gerando um par de chaves:

```text
chave privada da CA
chave pública da CA
```

A chave privada é usada para assinar certificados. A chave pública é distribuída dentro dos certificados para que os clientes consigam verificar essas assinaturas.

No topo da hierarquia, uma root CA, isto é, a autoridade certificadora raiz, cria um certificado para si mesma:

```text
subject = Root CA
issuer  = Root CA
```

Esse é um certificado self-signed, ou seja, assinado pela própria chave privada da CA que ele representa.

Depois, uma CA intermediária gera seu próprio par de chaves e cria um CSR (Certificate Signing Request) contendo a chave pública da própria CA intermediária. A root assina esse certificado intermediário.

Por fim, o servidor gera seu par de chaves, cria um CSR para o domínio desejado contendo a chave pública do próprio servidor e a CA intermediária assina o certificado leaf.

Um CSR não é um certificado pronto. Ele é um pedido de emissão que normalmente carrega:

- a chave pública do solicitante, que deverá entrar no certificado;
- o subject desejado;
- os SANs desejados;
- uma assinatura feita com a chave privada correspondente, para provar que o solicitante realmente possui aquela chave.

No caso de um certificado público para domínio, a CA ainda precisa validar o controle daquele domínio antes de emitir o leaf. A CA não coloca sua própria chave pública no leaf; ela coloca a chave pública do servidor e assina o certificado com a chave privada da CA.

O desenho fica assim:

```text
Root CA
  assina
Intermediate CA
  assina
Leaf certificate
```

No caso do Mercado Pago:

```text
Amazon Root CA 1
  assina
Amazon RSA 2048 M04
  assina
www.mercadopago.com.br
```

Em termos criptográficos:

```text
assinatura do leaf         -> feita com a chave privada do intermediate
assinatura do intermediate -> feita com a chave privada da root
```

Na validação, o cliente faz o caminho inverso:

```text
usa a chave pública do intermediate para validar o leaf
usa a chave pública da root para validar o intermediate
```

Mas a validação não para nas assinaturas. O cliente também precisa verificar:

- se o hostname acessado aparece no SAN do leaf;
- se os certificados estão dentro do período de validade;
- se os certificados de CA possuem extensões compatíveis com o papel de CA;
- se a cadeia respeita as restrições de path building, isto é, se cada certificado pode ocupar aquele papel na cadeia e assinar o próximo;
- se o caminho termina em uma trust anchor aceita localmente.

Eu volto a essas regras depois de resolver a pergunta principal da PoC: qual caminho o OpenSSL realmente escolheu para validar essa cadeia?

## Pergunta 5: o servidor enviou a root final?

À primeira vista, `cert-03.pem` parece uma root porque seu `subject` é `Amazon Root CA 1`.

Mas ele não é uma root self-signed.

Veja novamente:

```text
subject=C=US, O=Amazon, CN=Amazon Root CA 1
issuer=C=US, ST=Arizona, L=Scottsdale, O=Starfield Technologies, Inc., CN=Starfield Services Root Certificate Authority - G2
```

Se fosse self-signed, eu esperaria:

```text
subject = Amazon Root CA 1
issuer  = Amazon Root CA 1
```

Mas o `issuer` é a Starfield.

Isso já levanta uma nova pergunta:

```text
Como um certificado pode representar a Amazon Root CA 1 e, ao mesmo tempo, ser assinado pela Starfield?
```

Mas, por enquanto, a resposta da pergunta atual já está no `issuer`: o último certificado enviado pelo servidor ainda aponta para outra CA. Portanto, ele não é a root self-signed final.

A resposta é:

```text
o servidor não enviou a root final que o OpenSSL usou como trust anchor, isto é, como o ponto final já confiado localmente para encerrar a validação
```

## Pergunta 6: se o servidor não enviou a root final, como a validação passou?

Agora eu junto os certificados auxiliares enviados pelo servidor:

```bash
cat cert-02.pem cert-03.pem > intermediates.pem
```

E peço ao OpenSSL para validar o leaf:

```bash
openssl verify \
  -show_chain \
  -verify_hostname mercadopago.com.br \
  -untrusted intermediates.pem \
  cert-01.pem
```

Saída:

```text
cert-01.pem: OK
Chain:
depth=0: CN=www.mercadopago.com.br (untrusted)
depth=1: C=US, O=Amazon, CN=Amazon RSA 2048 M04 (untrusted)
depth=2: C=US, O=Amazon, CN=Amazon Root CA 1
```

Essa saída é extremamente interessante.

O leaf e o intermediate aparecem como `untrusted` porque foram fornecidos ao OpenSSL como material auxiliar para construir a cadeia, e não como trust anchors. Neste experimento, esse material veio do servidor. Isso não significa que sejam maliciosos.

Já o `depth=2`, `Amazon Root CA 1`, não aparece como `untrusted`.

Isso indica que o OpenSSL encontrou localmente uma `Amazon Root CA 1` confiável e encerrou a cadeia ali.

Em outras palavras:

```text
o servidor enviou:
Leaf -> Amazon RSA 2048 M04 -> Amazon Root CA 1 cross-signed

o OpenSSL validou:
Leaf -> Amazon RSA 2048 M04 -> Amazon Root CA 1 local
```

Eu ainda preciso explicar com calma o que significa `cross-signed`. Por enquanto, o ponto importante é que o certificado enviado pelo servidor não foi o certificado que encerrou a validação.

Essa é uma das ideias mais importantes de todo o artigo:

```text
a cadeia apresentada pelo servidor não precisa ser idêntica à cadeia escolhida pelo cliente
```

O servidor entrega material para o cliente construir um caminho.

O cliente escolhe o caminho que consegue validar até uma trust anchor local.

## Pergunta 7: onde essa trust anchor local vivia?

No começo da PoC, o comando `openssl version -a` já havia mostrado que o OpenSSL instalado via Homebrew usava:

```text
OPENSSLDIR: "/opt/homebrew/etc/openssl@3"
```

Agora essa pista passa a importar. Dentro desse diretório, o bundle de CAs confiáveis usado por esse OpenSSL estava em:

```text
/opt/homebrew/etc/openssl@3/cert.pem
```

Eu torno a validação explícita usando esse bundle diretamente:

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

O resultado continuou sendo `OK`, confirmando que a trust anchor usada pelo OpenSSL vinha daquele bundle local.

Esse ponto é importante porque não existe um trust store universal.

Vale separar os dois termos. Trust store é o conjunto local de CAs confiáveis disponíveis para o cliente. Trust anchor é o certificado ou a chave específica desse conjunto que encerra uma validação concreta.

| Ambiente | Trust store principal |
|---|---|
| macOS nativo | Keychain |
| OpenSSL Homebrew | Bundle próprio em `cert.pem` |
| Android | System store, user store e anchors da aplicação |
| Firefox | Store próprio baseado em NSS |
| Java | `cacerts` |

Uma cadeia pode ser válida no navegador e inválida no OpenSSL. Pode ser válida no macOS e inválida em uma JVM. Pode ser aceita pelo Android de sistema e rejeitada por uma aplicação que embute suas próprias anchors.

Trust anchor é sempre uma decisão local do cliente.

Também vale guardar uma nuance importante: trust anchor é um papel na validação, não apenas um tipo de certificado. A `Amazon Root CA 1` self-signed local atuou como trust anchor no caminho que eu observei. Já a `Amazon Root CA 1` cross-signed enviada pelo servidor, no caminho legado, atuaria como um certificado intermediário adicional até a Starfield.

## Pergunta 8: como provar que havia uma Amazon Root CA 1 local?

Eu extraio os certificados do bundle do OpenSSL:

```bash
mkdir -p roots

awk '
/-----BEGIN CERTIFICATE-----/ {n++; file=sprintf("roots/root-%03d.pem", n)}
file {print > file}
/-----END CERTIFICATE-----/ {close(file); file=""}
' /opt/homebrew/etc/openssl@3/cert.pem
```

Depois eu procuro pela `Amazon Root CA 1`:

```bash
for cert in roots/root-*.pem; do
  subject=$(openssl x509 -in "$cert" -noout -subject)
  if echo "$subject" | grep -q "Amazon Root CA 1"; then
    echo "$cert"
    openssl x509 -in "$cert" -noout -subject -issuer -serial -dates
  fi
done
```

Eu encontro:

```text
roots/root-022.pem
subject=C=US, O=Amazon, CN=Amazon Root CA 1
issuer=C=US, O=Amazon, CN=Amazon Root CA 1
```

Essa era a versão self-signed local da `Amazon Root CA 1`.

Aqui aparece uma distinção importante:

```text
uma root não é confiável porque é self-signed;
ela é confiável porque foi adicionada ao trust store do cliente.
```

Qualquer pessoa pode gerar um certificado self-signed. O que transforma uma root em trust anchor é a decisão local do sistema, runtime ou aplicação de confiar nela.

## Pergunta 9: como a Amazon Root CA 1 pode existir em dois certificados diferentes?

Agora eu chego ao ponto mais sutil da PoC.

Eu comparo o certificado enviado pelo servidor com a root local:

```bash
echo "===== cert-03.pem enviado pelo servidor ====="
openssl x509 -in cert-03.pem -noout -subject -issuer -fingerprint -sha256

echo
echo "===== root local ====="
openssl x509 -in roots/root-022.pem -noout -subject -issuer -fingerprint -sha256
```

O resultado conceitual foi:

| Campo | `cert-03.pem` | `roots/root-022.pem` |
|---|---|---|
| Subject | Amazon Root CA 1 | Amazon Root CA 1 |
| Issuer | Starfield Services Root Certificate Authority - G2 | Amazon Root CA 1 |
| Fingerprint | Diferente | Diferente |

Isso prova que são certificados diferentes.

Fingerprint é o hash do certificado inteiro. Se qualquer detalhe do certificado mudar, como issuer, número serial, validade ou assinatura, o fingerprint também muda.

Mas ainda faltava responder:

```text
Se são certificados diferentes, por que ambos servem para validar a mesma CA?
```

## Pergunta 10: o que é igual entre eles?

Eu comparo o hash do SPKI (Subject Public Key Info) de ambos:

```bash
echo "===== hash do SPKI cert-03.pem ====="
openssl x509 -in cert-03.pem -pubkey -noout \
  | openssl pkey -pubin -outform DER \
  | openssl dgst -sha256

echo "===== hash do SPKI root local ====="
openssl x509 -in roots/root-022.pem -pubkey -noout \
  | openssl pkey -pubin -outform DER \
  | openssl dgst -sha256
```

Saída:

```text
SHA2-256(stdin)= fbe3018031f9586bcbf41727e417b7d1c45c2f47f93be372a17b96b50757d5a2
SHA2-256(stdin)= fbe3018031f9586bcbf41727e417b7d1c45c2f47f93be372a17b96b50757d5a2
```

O fingerprint do certificado inteiro era diferente.

O hash do SPKI era igual.

Isso significa que os dois certificados carregavam a mesma chave pública.

SPKI é a estrutura do certificado que descreve a chave pública: qual algoritmo ela usa e quais são os bytes da chave em si.

Essa é a diferença fundamental:

```text
fingerprint  = hash do certificado inteiro
hash do SPKI = hash da estrutura da chave pública
```

Se qualquer campo do certificado mudar, o fingerprint muda:

- issuer;
- serial;
- validade;
- extensões;
- assinatura.

Mas se a chave pública continuar a mesma, o SPKI continua igual.

## Pergunta 11: o que significa cross-signed, afinal?

Uma root self-signed da Amazon Root CA 1 tem:

```text
subject = Amazon Root CA 1
issuer  = Amazon Root CA 1
```

Ela é um certificado emitido pela própria Amazon Root CA 1 para a sua própria chave pública.

Já a versão cross-signed tem:

```text
subject = Amazon Root CA 1
issuer  = Starfield Services Root Certificate Authority - G2
```

Ela carrega a mesma chave pública da Amazon Root CA 1, mas foi assinada pela Starfield.

O processo conceitual é:

```text
1. Amazon gera o par de chaves da Amazon Root CA 1.
2. Amazon cria um certificado self-signed para essa chave.
3. Para obter uma versão cross-signed, a Amazon prepara uma solicitação de certificação para a Starfield, normalmente representada por um CSR.
4. Esse CSR carrega a chave pública da Amazon Root CA 1 e é assinado com a chave privada correspondente, provando que a Amazon controla aquela chave.
5. A Starfield usa essa solicitação para emitir outro certificado para a mesma chave pública da Amazon Root CA 1.
6. Esse segundo certificado é assinado pela Starfield.
```

Aqui vale um cuidado de linguagem: não é o certificado que "gera" o CSR. É a entidade operadora da CA, neste caso a Amazon, que cria essa solicitação usando a chave privada associada à Amazon Root CA 1. O ponto conceitual importante é que a Starfield não recebe a chave privada da Amazon; ela recebe uma solicitação que contém a chave pública e uma prova de posse da chave privada correspondente.

Assim surgem dois certificados diferentes:

```text
Amazon Root CA 1 self-signed
Amazon Root CA 1 cross-signed
```

Neste caso, eles representam a mesma CA lógica porque têm o mesmo subject, carregam a mesma chave pública e ambos são certificados de CA.

Eles são certificados diferentes porque foram emitidos em contextos diferentes e assinados por autoridades diferentes.

## Pergunta 12: por que alguém faria isso?

Para compatibilidade.

Quando a Amazon Root CA 1 surgiu, clientes modernos podiam receber essa root em seus trust stores e confiar nela diretamente.

Mas clientes antigos talvez ainda não tivessem essa root instalada.

Esses clientes antigos já confiavam em roots mais velhas, como a Starfield.

Então o cross-signing cria uma ponte:

```text
Cliente moderno:
Leaf -> Amazon RSA 2048 M04 -> Amazon Root CA 1 local

Cliente legado:
Leaf -> Amazon RSA 2048 M04 -> Amazon Root CA 1 cross-signed -> Starfield Root
```

No cliente moderno, a validação termina cedo, porque ele já conhece a Amazon Root CA 1.

No cliente legado, a validação continua até a Starfield, porque é nela que ele confia localmente.

## Pergunta 13: por que o cliente moderno consegue encerrar na root local em vez de continuar pela versão cross-signed?

Porque a assinatura do intermediate foi feita pela chave privada da Amazon Root CA 1.

Para verificar essa assinatura, o cliente precisa da chave pública correspondente.

Tanto a versão self-signed quanto a versão cross-signed carregam a mesma chave pública.

Logo:

```text
chave pública da Amazon Root CA 1 self-signed
==
chave pública da Amazon Root CA 1 cross-signed
```

É por isso que ambas conseguem validar a assinatura do `Amazon RSA 2048 M04`.

O cliente não substitui literalmente um certificado por outro. O path builder encontra uma candidata local confiável com o mesmo subject e a mesma chave pública e constrói um caminho alternativo até ela.

O cliente moderno não precisa continuar até a Starfield porque já possui localmente uma trust anchor da Amazon Root CA 1 que carrega aquela mesma chave pública.

Já o cliente legado usa o certificado cross-signed como um intermediário adicional:

```text
1. valida o leaf com a chave pública do Amazon RSA 2048 M04;
2. valida o Amazon RSA 2048 M04 com a chave pública da Amazon Root CA 1;
3. valida a Amazon Root CA 1 cross-signed com a chave pública da Starfield;
4. encerra a cadeia na Starfield local.
```

## Uma última camada: o que mais o cliente valida?

Até aqui, eu resolvi o mistério central da PoC: o servidor enviou uma cadeia, mas o OpenSSL construiu outra até uma trust anchor local compatível.

Ainda assim, vale voltar a uma observação que ficou aberta na pergunta 4. Uma cadeia não é aceita apenas porque as assinaturas batem. O cliente também verifica se cada certificado pode ocupar o papel que está ocupando.

Uma das extensões mais importantes para isso é `basicConstraints`.

Se eu quiser inspecioná-la nos certificados capturados, posso usar:

```bash
openssl x509 -in cert-01.pem -noout -ext basicConstraints
openssl x509 -in cert-02.pem -noout -ext basicConstraints
```

O que eu procuro nessa saída é a diferença entre:

```text
CA:FALSE
```

e:

```text
CA:TRUE
```

Um leaf normalmente aparece com `CA:FALSE`. Isso significa que ele pode representar um servidor, mas não pode ser aceito como uma CA capaz de assinar outro certificado abaixo dele em uma cadeia válida.

Um intermediate normalmente aparece com `CA:TRUE`. Isso significa que ele pode ser aceito como autoridade certificadora e assinar certificados leaf.

Essa distinção impede que alguém pegue um certificado comum de servidor e tente usá-lo como se fosse uma CA intermediária. A chave privada correspondente ainda conseguiria produzir uma assinatura criptográfica, mas a cadeia deveria ser rejeitada porque aquele certificado não tem permissão para atuar como CA.

Além disso, uma CA pode carregar uma restrição como `pathlen:0`. Nesse caso, ela pode assinar certificados leaf, mas não pode assinar outro intermediate abaixo dela. Essa é uma forma de limitar a profundidade da cadeia e impedir que a hierarquia cresça indefinidamente.

Outra extensão relacionada é `keyUsage`. Mesmo que um certificado tenha `CA:TRUE`, ele normalmente também precisa permitir assinatura de certificados para atuar como CA.

Por fim, este artigo não cobre mecanismos de revogação, como CRL (Certificate Revocation List) e OCSP (Online Certificate Status Protocol). Eles formam outra parte importante da validação em alguns clientes, mas não eram necessários para responder a pergunta central desta PoC.

## O que o experimento realmente me ensinou

Ao final da PoC, eu consegui provar:

1. o servidor enviou um leaf, um intermediate e um certificado auxiliar;
2. o leaf era válido para `mercadopago.com.br` porque o SAN continha esse hostname;
3. o certificado auxiliar enviado pelo servidor era uma versão cross-signed da Amazon Root CA 1;
4. o OpenSSL não encerrou a validação nesse certificado enviado pelo servidor;
5. o OpenSSL encontrou localmente uma Amazon Root CA 1 self-signed;
6. a versão local e a versão cross-signed tinham fingerprints diferentes;
7. ambas tinham o mesmo hash de SPKI;
8. portanto, eram certificados diferentes representando a mesma CA lógica.

## Modelo mental final

Depois de percorrer o experimento inteiro, a forma mais útil de pensar em TLS é:

```text
1. O servidor apresenta certificados.
2. O cliente verifica o hostname no SAN do leaf.
3. O cliente valida assinatura por assinatura.
4. O cliente tenta construir um caminho até uma trust anchor local.
5. A cadeia usada pelo cliente pode ser diferente da cadeia apresentada pelo servidor.
```

E a ideia mais importante de todas:

```text
Certificado não é a mesma coisa que chave pública.
```

Um certificado é um documento assinado.

Uma chave pública é o material criptográfico que esse documento associa a uma entidade.

Por isso, dois certificados podem:

- ter issuers diferentes;
- ter números seriais diferentes;
- ter fingerprints diferentes;
- e, quando preservam a mesma chave pública e continuam sendo certificados de CA, ainda assim representar a mesma autoridade certificadora.

Em TLS, confiar em um domínio não significa confiar em um arquivo isolado. Significa verificar que o hostname pedido aparece no leaf e confiar em um caminho de assinaturas que termina em uma trust anchor já aceita localmente pelo cliente.

## Referência rápida dos termos usados

| Termo | Significado |
|---|---|
| HTTPS (Hypertext Transfer Protocol Secure) | HTTP transportado sobre uma conexão TLS |
| Handshake | Negociação inicial em que cliente e servidor combinam parâmetros e validam identidade |
| TLS (Transport Layer Security) | Protocolo que protege confidencialidade, integridade e autenticidade da conexão |
| PKI (Public Key Infrastructure) | Infraestrutura que organiza chaves, certificados e autoridades certificadoras |
| CA (Certificate Authority) | Autoridade certificadora que assina certificados |
| SNI (Server Name Indication) | Extensão do TLS que informa ao servidor qual hostname o cliente quer acessar |
| PEM (Privacy-Enhanced Mail) | Formato textual usado para representar certificados e chaves |
| Leaf | Certificado do servidor final, mais próximo do domínio acessado |
| Intermediate | CA intermediária usada para assinar certificados leaf |
| Root CA | CA no topo da hierarquia de confiança |
| CN (Common Name) | Atributo histórico de nome dentro do certificado |
| SAN (Subject Alternative Name) | Extensão que lista os hostnames válidos para o certificado |
| CSR (Certificate Signing Request) | Pedido de emissão que carrega a chave pública do solicitante e os atributos desejados para o certificado |
| basicConstraints | Extensão que indica se um certificado pode atuar como CA e, quando presente, limita quantas CAs adicionais podem existir abaixo dele |
| keyUsage | Extensão que indica para quais operações criptográficas a chave do certificado pode ser usada |
| CRL (Certificate Revocation List) | Lista publicada por uma CA com certificados revogados |
| OCSP (Online Certificate Status Protocol) | Protocolo para consultar o status de revogação de um certificado |
| Trust store | Conjunto local de CAs confiáveis disponíveis para o cliente |
| Trust anchor | Certificado ou chave confiável que encerra uma validação concreta |
| SPKI (Subject Public Key Info) | Estrutura do certificado que descreve a chave pública |
| Fingerprint | Hash do certificado inteiro |
| pathlen | Restrição em um certificado de CA que limita quantas CAs intermediárias adicionais podem existir abaixo dele na cadeia |
| Path building | Processo pelo qual o cliente monta um caminho de certificados, do leaf até uma trust anchor local, a partir do material disponível |
| Self-signed | Certificado assinado pela própria chave privada da entidade que representa |
| Cross-signed | Certificado de uma CA assinado por outra CA |
