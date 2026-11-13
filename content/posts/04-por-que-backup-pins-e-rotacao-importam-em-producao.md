+++
title = "Por que backup pins e rotação importam em produção"
translationKey = "android-pinning-backup-pins-rotation"
date = "2026-11-13"
author = "Thiago Siqueira"
tags = ["android", "ssl-pinning", "mobile", "pki", "operacoes"]
keywords = ["android", "certificate pinning", "backup pin", "key rotation", "network security config"]
description = "Reproduzo a rotação da chave do servidor para mostrar por que um cliente que só conhece o pin antigo perde a conexão, enquanto outro preparado com um backup pin continua funcionando, e o que essa continuidade exige da operação."
showFullContent = false
readingTime = true
hideComments = false
+++

Uma app consegue conversar com seu servidor, recebe a resposta esperada e aplica corretamente
certificate pinning. Algum tempo depois, a equipe substitui o certificado TLS do servidor.
O domínio continua o mesmo, o novo certificado é válido e a mudança é legítima. Ainda assim,
a app deixa de conectar.

O que ela está rejeitando?

Nos artigos anteriores, eu investigo a confiança em TLS (Transport Layer Security), separo
falhas de trust store de falhas de pinning e mostro como uma verificação local pode ser
identificada e alterada. Agora eu quero olhar para o controle durante uma mudança normal do
serviço, enquanto a app continua executando sua política original.

A pergunta que fecha a série é: *como trocar a chave legítima do servidor sem perder a conexão com os clientes já instalados?*

Para responder, eu acompanho uma troca de chave e comparo o comportamento de dois clientes.
O experimento fornece a evidência, mas a questão é mais ampla: uma decisão de confiança gravada
na app precisa continuar compatível com um servidor que vai mudar ao longo do tempo.

## A conexão funciona; o que exatamente a app espera encontrar?

Eu começo com uma app Android que consulta um serviço HTTPS (Hypertext Transfer Protocol
Secure) e recebe uma resposta com sucesso. O certificado do servidor cobre o domínio acessado,
está dentro do período de validade e sua cadeia chega a uma autoridade certificadora aceita
pela app. Vou chamar essa autoridade de CA (Certificate Authority).

Além dessas verificações, o cliente aplica pinning à chave pública do certificado final do
servidor, também chamado de leaf. Chamo essa chave de **A**.

O pin configurado na app é um hash de SPKI (Subject Public Key Info), a estrutura que descreve
a chave pública no certificado. Na implementação usada no estudo, esse hash é calculado com
SHA-256. A app não guarda a chave privada do servidor: guarda um valor que permite reconhecer
a chave pública esperada.

Isso acrescenta uma condição à conexão. A cadeia pode ser confiável e o hostname correto,
mas uma chave diferente ainda pode ser rejeitada pelo pinning.

Enquanto o servidor apresenta um certificado com A, as duas condições estão satisfeitas:
a validação TLS normal aceita o certificado, e a verificação adicional encontra o pin esperado.
A app recebe `HTTP 200`, o código de uma resposta HTTP bem-sucedida.

Até aqui, o controle está funcionando. Para entender o que pode acontecer numa manutenção,
preciso primeiro separar duas mudanças que costumam ser tratadas como equivalentes: trocar o
certificado e trocar a chave.

## Pergunta 1: um certificado novo precisa ter uma chave nova?

No primeiro artigo, eu comparo certificados diferentes que carregam a mesma chave pública.
Essa distinção também é importante numa renovação.

O certificado é um documento assinado: contém a chave pública, os nomes para os quais foi
emitido, seu período de validade e outros campos. O fingerprint é o hash desse documento
inteiro. O pin de SPKI que estou estudando corresponde à estrutura da chave pública.

Se um certificado for reemitido preservando o mesmo SPKI, seu serial, validade e assinatura
podem mudar, assim como o fingerprint. O pin dessa chave, porém, continua correspondendo.
A conexão ainda dependerá de o novo certificado e sua cadeia passarem pelas demais validações.

Eu confiro essa distinção antes de trocar de chave. Primeiro, o servidor usa o certificado
original com A. Depois, apresento um certificado reemitido com a mesma chave, mas com outro
serial e outro período de validade. O cliente continua aceitando apenas o pin de A.

O fingerprint SHA-256 muda de `92:41:74:CF…` para `17:2D:64:C1…`, enquanto o hash de SPKI
permanece igual. Nos dois casos, a app recebe `HTTP 200`.

| Evidência observada | Certificado original | Certificado reemitido |
| --- | --- | --- |
| Fingerprint SHA-256, abreviado | `92:41:74:CF…` | `17:2D:64:C1…` |
| Chave pública | A | A |
| Cliente que aceita apenas A | HTTP 200 | HTTP 200 |

Os [valores completos e certificados públicos](#TODO-link-github-evidence-article-04-certificate-comparison)
permitem conferir a comparação. A reemissão mudou o documento, mas preservou a correspondência
do pin. Por isso, a frase “a renovação quebrou o pinning” precisa de uma pergunta adicional:
**a chave pública também mudou?**

| Mudança | O que acontece com o pin da chave do servidor |
| --- | --- |
| Certificado reemitido com o mesmo SPKI | O hash de SPKI continua correspondendo |
| Certificado emitido com outra chave pública | O cliente precisa conhecer um pin compatível com essa outra chave |
| Certificado vencido ou inválido para o domínio | Um pin correspondente não dispensa a validação normal do certificado |

Para investigar a segunda situação, eu substituo A por outro par de chaves. Chamo a nova chave
pública de **B**. O servidor recebe um certificado que contém B e usa a chave privada
correspondente durante a conexão.

O novo certificado continua válido para o mesmo domínio e é emitido pela mesma CA. O que
precisa mudar para o pinning é o hash da chave pública apresentada.

## Pergunta 2: por que a app rejeita um servidor que continua legítimo?

Depois da troca, o cliente que conhece apenas A deixa de receber a resposta. A tela mostra
uma falha, com `Exception: SSLHandshakeException` e causa raiz `CertificateException: Pin verification failed`.

<img src="/images/posts/04-por-que-backup-pins-e-rotacao-importam-em-producao/profile-b-nscPinnedPrimaryOnly.png" alt="Cliente que conhece apenas A mostrando FAILURE em vermelho e Pin verification failed depois da troca legítima para a chave B" class="center" width="400">

*Após a troca legítima para B, o cliente que conhece apenas A rejeita a conexão por incompatibilidade dos pins.*

Esse resultado lembra a rejeição do proxy estudada no segundo artigo. Mas, desta vez, a conexão
vai diretamente ao servidor legítimo. Não há um intermediário trocando o certificado no caminho.

Eu confiro os dois certificados: ambos continuam válidos para o hostname e para o uso como
servidor TLS, com uma cadeia que termina na CA aceita pelos clientes. A diferença relevante para a regra
da app é a chave pública.

O servidor agora apresenta B. O cliente ainda espera A.

- **Antes da troca:** cliente espera A → servidor apresenta A → conexão aceita.
- **Depois da troca:** cliente espera A → servidor apresenta B → pinning rejeita.

A CA afirma que o novo certificado é válido para aquele domínio. O cliente, por sua vez,
continua aplicando uma restrição mais específica: só aceita uma cadeia que corresponda aos
pins que recebeu.

Essa regra não consegue distinguir a intenção de quem fez a mudança. Uma chave inesperada
pode vir de uma interceptação ou de uma manutenção legítima. Em ambos os casos, a correspondência
esperada está ausente.

É por isso que configurar o pinning corretamente não basta para garantir continuidade. Ao
distribuir o pin de A, eu também crio uma dependência entre a app já instalada e a configuração
futura do servidor.

## Pergunta 3: essa interrupção poderia ter sido evitada?

Para responder, eu comparo o resultado com outro cliente. Ele consulta o mesmo serviço e faz
as mesmas verificações de certificado, mas já conhece duas alternativas: os pins de A e de B.
Essa configuração estava instalada antes da troca no servidor.

Enquanto o servidor ainda usa A, os dois clientes funcionam. Depois da promoção de B, o
comportamento se divide:

| Política instalada no cliente | Servidor com A | Servidor com B |
| --- | --- | --- |
| Aceita apenas o pin de A | Conecta | Rejeita por pinning |
| Aceita os pins de A e B | Conecta | Conecta |

Na pergunta anterior, o cliente que conhece apenas A rejeitou a conexão. A captura abaixo
mostra o cliente que já conhece A e B, conectado diretamente ao mesmo servidor depois da mesma
troca legítima.

<img src="/images/posts/04-por-que-backup-pins-e-rotacao-importam-em-producao/profile-b-nscPinned.png" alt="Cliente que já conhece A e B recebendo HTTP 200 depois da mesma troca de chave no servidor" class="center" width="400">

*O cliente continua conectado porque B já era uma alternativa aceita pela política instalada.*

O segundo cliente continua recebendo `HTTP 200`. Eu não preciso remover a verificação de pins
nem instalar uma nova CA entre os dois estados. B satisfaz uma alternativa que a app já tinha
recebido.

É essa alternativa preparada antecipadamente que eu chamo de **backup pin**.

A palavra “backup” pode sugerir um mecanismo que fica desligado até a chave principal falhar.
Na configuração usada aqui, o conjunto de pins funciona como uma lista de alternativas aceitas.
Não há uma prioridade que obrigue o Android a tentar A antes de permitir B.

O pin de B já participa da decisão desde que a política esteja em uso. Seu papel de backup
vem do planejamento: o servidor ainda usa A, mas o cliente está preparado para aceitar B.
O código da plataforma aplica essa correspondência em
[`NetworkSecurityTrustManager.checkPins`](https://github.com/aosp-mirror/platform_frameworks_base/blob/master/core/java/android/security/net/config/NetworkSecurityTrustManager.java).

Também não é necessário que o servidor apresente as duas chaves ao mesmo tempo para esse
experimento funcionar. Ele passa de um certificado com A para outro com B; a política do
cliente aceita as duas possibilidades.

Aqui, os pins correspondem às chaves do leaf. Fixar a chave de uma CA mudaria o escopo da
restrição: outros certificados com novas chaves de servidor poderiam continuar compatíveis
com aquela CA. A comparação deste artigo mantém o pin na chave do servidor para tornar o
efeito da troca explícito.

## Pergunta 4: um segundo hash é suficiente para ter um backup?

O resultado mostra o valor de aceitar B antecipadamente. Mas ele abre outra pergunta:
se eu colocar qualquer hash adicional na app, já tenho uma alternativa utilizável?

Para aceitar a conexão, a app precisa encontrar uma correspondência com um pin. Para oferecer
essa conexão, o servidor precisa de mais coisas: a chave privada de B e um certificado válido
que contenha a chave pública correspondente.

Se eu configurar um hash sem ter acesso à chave privada associada, não consigo simplesmente
transformá-lo numa identidade utilizável pelo servidor. O mesmo problema aparece se eu tenho
a chave, mas não consigo obter ou instalar seu certificado quando preciso fazer a troca.

No experimento, B é um par de chaves separado de A. O certificado de B é emitido usando a chave
pública que corresponde ao pin de backup já configurado. A comparação do SPKI confirma essa
relação antes da promoção.

Essa ordem importa: se eu gerar outra chave na hora da mudança, ela terá outro SPKI e não
corresponderá ao backup que distribuí.

Portanto, preparar o backup envolve duas responsabilidades relacionadas. A política do cliente
precisa aceitar B, e a operação do servidor precisa conseguir usar B. Isso inclui preservar
a chave privada de forma adequada, controlar seu acesso e ter um caminho viável para emissão
e instalação do certificado.

O experimento local demonstra a correspondência entre chave e pin. Em um serviço real, a
capacidade de executar essa troca precisa ser sustentada pela gestão das chaves e dos certificados.

## Pergunta 5: o backup protege também quem ainda não atualizou a app?

O cliente com A e B sobrevive à mudança porque recebeu o pin de B antes dela. Essa observação
impõe um limite importante: o backup só ajuda as instalações que já o conhecem.

Imagine que uma versão antiga tenha sido distribuída com apenas A. Depois, eu publico uma
versão que aceita A e B. Enquanto o servidor continua em A, ambas conseguem conectar. Esse
período permite distribuir a nova configuração sem exigir imediatamente a nova chave.

Quando o servidor passa a B, porém, a versão antiga continua esperando apenas A. Publicar
a atualização não altera a política das instalações que ainda não a receberam.

Há usuários que adiam atualizações, dispositivos que ficam offline e versões que deixam de
receber novas releases. A decisão de promover B precisa considerar quais desses clientes
continuam suportados. Não existe um número universal de dias que garanta a atualização de todos.

Eu passo a enxergar a rotação como uma mudança coordenada entre dois lados: o servidor consegue
adotar B rapidamente, mas o conhecimento de B precisa chegar a uma população de clientes com
ritmos diferentes de atualização.

É essa diferença de ritmo que transforma o backup pin em algo operacionalmente importante.
Ele abre uma janela de compatibilidade. Não garante, sozinho, que toda a base esteja dentro dela.

## Pergunta 6: se B virou a chave atual, o que acontece na próxima troca?

Depois de promover B, eu poderia manter a política A+B indefinidamente. Ela continuaria
aceitando a chave atual, mas não prepararia uma futura chave C.

Para repetir a transição com continuidade, eu preciso distribuir uma política que aceite B
e C enquanto o servidor ainda usa B. A mesma lógica que permitiu a primeira troca passa a
orientar a próxima:

| Momento | Chave usada pelo servidor | Política que preparo nos clientes |
| --- | --- | --- |
| Antes da primeira troca | A | Aceitar A e B |
| Depois de promover B | B | Distribuir uma versão que aceite B e C |
| Depois de preparar os clientes para C | C | Manter uma alternativa para a próxima mudança |

Durante o período em que o servidor usa B, clientes A+B e B+C têm uma chave aceita em comum.
Isso permite atualizar a configuração sem interromper imediatamente as conexões de quem ainda
está na versão anterior.

Mas, quando o servidor passa a C, clientes que ficaram em A+B voltam a enfrentar o mesmo
problema. A continuidade depende de distribuir a próxima alternativa antes de exigi-la.

Há ainda uma consequência para a remoção de chaves antigas. Trocar o servidor para B não remove
A da política A+B que já está instalada. O pin antigo permanece uma alternativa reconhecida
por aquela configuração, sujeito às demais validações do certificado.

Se a motivação da troca for um comprometimento de A, isso importa. Deixar de usar uma chave
no servidor e deixar de aceitá-la nos clientes são mudanças diferentes. Atualização da política,
validação de certificados e procedimentos de resposta precisam ser considerados em conjunto.

Assim, o planejamento não termina quando B entra em uso. Ele precisa incluir a próxima chave
e o destino das chaves que deixaram de ser desejáveis.

## Pergunta 7: expirar os pins resolve a incompatibilidade?

Existe outro mecanismo que pode parecer uma saída para clientes que não se atualizam: a
expiração da política de pinning.

No Android, o Network Security Config (NSC) permite definir uma data no atributo `expiration`
do conjunto de pins. Depois dela, aqueles pins deixam de ser aplicados; sem esse atributo,
o conjunto não recebe essa data de expiração. A
[referência do NSC](https://developer.android.com/privacy-and-security/security-config#pin-set)
documenta esse comportamento.

Essa expiração é diferente da validade do certificado do servidor. Se o certificado continua
válido, mas a conexão falhava apenas porque nenhum pin correspondia, deixar de aplicar os
pins pode permitir que ela volte a funcionar. Se o certificado estiver vencido ou não representar
o domínio acessado, a expiração dos pins não corrige esse outro problema.

Isso cria uma escolha entre continuidade e duração da restrição adicional. Um cliente que
nunca recebe novos pins pode deixar de exigir essa correspondência depois da data configurada.
Ao mesmo tempo, ele perde a proteção adicional que os pins ofereciam.

Eu não trato a expiração como renovação automática nem como substituto para distribuir as
próximas chaves. Ela precisa fazer parte da política de manutenção: por quanto tempo a equipe
quer exigir essa restrição num cliente que permaneceu com a configuração antiga?

## Pergunta 8: o que muda quando a troca precisa ser emergencial?

Na transição planejada, B já está preparada e seu pin já foi distribuído. Uma emergência pode
reduzir esse espaço de escolha. A chave de backup pode estar indisponível, a emissão do
certificado pode demorar ou a mudança necessária pode exigir outra chave.

O pin adicional não elimina essas dependências. Para contar com B como recuperação, eu preciso
saber se consigo colocá-la em uso dentro do tempo necessário e quais clientes conseguirão
reconhecê-la.

Também preciso pensar no caminho de volta. Se uma troca planejada apresentar um problema,
retornar temporariamente a A pode ser uma possibilidade. Se A foi comprometida, esse mesmo
retorno pode contrariar o objetivo da mudança. “Temos rollback” só é uma resposta útil quando
sei qual estado anterior continua aceitável.

Outro cuidado aparece quando considero atualizar os pins remotamente. Se a app precisa buscar
a nova configuração pelo mesmo endpoint cuja chave ela está rejeitando, pode ficar sem o canal
necessário para receber a correção. Uma solução de configuração remota exigiria um projeto
próprio de autenticidade, integridade e recuperação.

Remover a validação de cadeia ou hostname para restabelecer a comunicação também muda o
problema: a conexão perde garantias básicas de autenticação do servidor. A recuperação precisa
preservar essas garantias e ter um caminho que os clientes afetados realmente consigam usar.

## Como eu perceberia essa falha numa app distribuída?

No experimento, a exceção está visível. Para um usuário, o mesmo evento pode aparecer apenas
como “não foi possível conectar”. A API pode continuar saudável e atendendo outros clientes,
enquanto uma versão específica da app passa a rejeitar seu certificado.

Por isso, olhar apenas a disponibilidade do servidor não basta para interpretar o incidente.
Eu preciso relacionar a versão do cliente, a política que ele recebeu e a chave efetivamente
apresentada pelo ponto de terminação TLS ao qual ele se conectou.

Uma mudança aplicada gradualmente em vários servidores também pode produzir falhas intermitentes:
o cliente aceita a chave servida por um deles e rejeita a de outro. Para investigar, preciso
saber se todos os caminhos de atendimento estão apresentando certificados compatíveis com as
versões ainda suportadas.

O canal de telemetria merece a mesma atenção. Se os eventos de erro dependem da conexão que
acabou de falhar, a ausência de registros não demonstra que os clientes estejam funcionando.
Pode ser justamente parte da perda de conectividade que eu preciso detectar.

Essas consequências ampliam o que significa operar o controle: saber quais chaves deveriam
ser aceitas, acompanhar a distribuição dessa política e conseguir reconhecer quando cliente
e servidor deixaram de ser compatíveis.

## Depois dessa investigação, quando vale adotar pinning?

O experimento mostra que uma troca legítima pode ser aceita por um cliente preparado e rejeitada
por outro que só conhece a chave anterior. O ganho de restringir a confiança vem acompanhado
da necessidade de manter essa configuração compatível com o serviço.

Esse custo ajuda a entender o alerta da documentação Android, que
[não recomenda certificate pinning para apps Android](https://developer.android.com/privacy-and-security/security-ssl#restricting-your-app-to-specific-certificates)
pelo risco de mudanças no servidor deixarem clientes sem conectividade. Quando a restrição é
usada, o documento destaca pins de backup e expiração entre os cuidados operacionais.

Para avaliar uma adoção, eu preciso partir da ameaça concreta que pretendo dificultar. Depois,
preciso verificar se controlo as chaves, os certificados, a terminação TLS e a distribuição
necessária para manter as versões suportadas funcionando.

O terceiro artigo mostra o limite diante de alguém que controla o processo ou altera o pacote
da app. Este último acrescenta a responsabilidade de manter o controle funcionando durante
mudanças legítimas. As duas perspectivas precisam participar da mesma decisão.

Autenticação, autorização e validações no backend continuam necessárias. Uma restrição adicional
no canal não transforma o cliente num ambiente confiável nem substitui esses controles.

## Para quem quiser reproduzir a investigação

A demonstração A → B foi executada com dois clientes Android e certificados emitidos por uma
CA local. O [guia opcional de reprodução](#TODO-link-github-reproducao-rotation)
reúne os comandos, nomes das variantes, preparação e resultados esperados. Nenhuma dessas
etapas é necessária para acompanhar a explicação deste artigo.

As [novas evidências](#TODO-link-github-evidence-article-04-readme) incluem as telas, os certificados
públicos e os registros da reemissão com a mesma chave A e da troca A → B. As discussões sobre
C, emergências, expiração e observabilidade desenvolvem suas consequências operacionais, sem
representar testes adicionais de produção.

## O que fecha a série

Eu começo a série investigando como um cliente decide confiar no certificado de um domínio.
Depois, separo a escolha das CAs da restrição adicional dos pins e acompanho os pontos em que
essa restrição pode ser aplicada e alterada.

Na última investigação, a pergunta passa a ser como manter essa confiança durante uma mudança
legítima do serviço. O cliente que conhece A e B continua conectado porque a alternativa chegou
antes de ser necessária. O cliente que conhece apenas A perde a conexão, embora o servidor
continue apresentando um certificado válido.

O modelo mental com que eu termino é:

- *O certificado pode mudar sem que a chave mude.*
- *Se a chave muda, o cliente precisa reconhecer uma alternativa compatível.*
- *O backup só ajuda quem recebeu esse pin antes da troca.*
- *A próxima alternativa precisa chegar antes de ser exigida pelo servidor.*

O pin distribuído hoje passa a fazer parte das condições para mudar o serviço amanhã. Justificar
a adoção de pinning exige entender a ameaça que ele dificulta, o atacante que ainda consegue
alterá-lo e a manutenção necessária para que seus próprios usuários continuem conectados.

## Referência rápida dos termos usados

| Termo | Significado neste artigo |
| --- | --- |
| TLS (Transport Layer Security) | Protocolo usado para proteger e autenticar a conexão |
| CA (Certificate Authority) | Autoridade certificadora aceita na validação da cadeia |
| Leaf | Certificado final apresentado para representar o servidor |
| SPKI (Subject Public Key Info) | Estrutura da chave pública cujo hash é usado nos pins |
| Fingerprint | Hash do certificado inteiro |
| NSC (Network Security Config) | Mecanismo declarativo de configuração de rede do Android |
| Backup pin | Pin de uma chave alternativa preparada antes de uma troca |
| Rotação A → B | Substituição da chave A pela chave B no servidor |
| Expiration | Data após a qual o NSC deixa de aplicar aquele conjunto de pins |
| Rollback | Retorno a um estado anterior quando ele ainda é aceitável |
