---
title: "13 | Proteção de VLANs com EAPS/ERPS – Parte 1"
date: 2024-08-26 00:00:00 -0300
categories: [Artigos]
tags: [VLAN, EAPS, ERPS, Networking]
image:
  path: assets/img/posts/post-13/capa.png
---

Redundância e proteção em redes de computadores são cruciais para garantir alta disponibilidade e continuidade dos serviços, minimizando o impacto de falhas e interrupções. Essas práticas asseguram que, em caso de problemas em um componente ou caminho da rede, outros mecanismos ou rotas possam assumir, evitando downtime e mantendo a operação.

O EAPS (Ethernet Automatic Protection Switching) e o ERPS (Ethernet Ring Protection Switching) são protocolos que ajudam a alcançar esse objetivo. Nesta primeira parte do artigo irei abordar mais sobre o EAPS e realizar alguns laboratórios práticos.

EAPS (Ethernet Automatic Protection Switching)
O EAPS é um protocolo desenvolvido pela Extreme Networks para fornecer proteção de VLANS em redes Ethernet em topologia Anel. É utilizado para criar uma topologia tolerante a falhas, configurando um caminho primário e um caminho secundário para cada VLAN. Isso garante que, em caso de falha no caminho primário, o tráfego possa ser rapidamente redirecionado através do caminho secundário, mantendo a continuidade dos serviços e minimizando interrupções na rede.

![](assets/img/posts/post-13/capa.png)

O EAPS utiliza um mecanismo de detecção de falhas para identificar rapidamente problemas no enlace ou dispositivo. Quando uma falha é detectada, o protocolo realiza uma troca automática para um caminho alternativo predefinido, garantindo a continuidade do tráfego. 

Um anel é formado configurando um Domínio. Cada domínio tem um único “nó mestre” e muitos “nós de trânsito”. Cada nó possui uma porta primária e uma porta secundária, ambas conhecidas por poder enviar tráfego de controle para o nó mestre. Em operação normal, a porta secundária no mestre é bloqueada para todas as VLANs protegidas.

Quando ocorre uma falha de enlace, os dispositivos que detectam o problema enviam uma mensagem de controle para o mestre, e o mestre desbloqueia a porta secundária e instrui os nós de trânsito a limpar seus bancos de dados de encaminhamento. Os próximos pacotes enviados pela rede podem então ser transmitidos e aprendidos pela (agora habilitada) porta secundária sem interrupção na rede.

O mesmo switch pode pertencer a múltiplos domínios e, portanto, a múltiplos anéis. No entanto, esses atuam como entidades independentes e podem ser controlados individualmente.

Resumindo, a VLAN de controle precisa “loopar” para completar a o processo do EAPS, caso a VLAN não “loope” mais, será habilitado a porta que está bloqueada.

Benefícios:
– Tempo de Recuperação Rápido: O EAPS é projetado para detectar e reagir a falhas em menos de 50 ms, garantindo uma rápida recuperação e mínima interrupção dos serviços.

– Simplicidade e Eficácia: O protocolo é relativamente simples de implementar e configurar, oferecendo uma solução eficaz para a proteção de enlaces Ethernet sem complexidade excessiva.

Frame do EAPS (RFC 3619)
![](assets/img/posts/post-13/01.png)

Valores de Estados do EAPS

![](assets/img/posts/post-13/02.png)

*IDLE (0)*

Significado: O estado IDLE indica que o protocolo EAPS está inativo ou não está realizando atualmente nenhum processo de proteção. Em outras palavras, o EAPS não está monitorando o estado dos links ou realizando a troca automática de proteção.

*COMPLETE (1)*

Significado: No estado COMPLETE, o protocolo EAPS terminou com sucesso o processo de comutação ou recuperação de proteção e a Vlan de controle está loopando conforme necessário. Isso significa que a proteção foi ativada.

Uso: Esse estado é mostrado nó Master.

*FAILED (2)*

Significado: O estado FAILED indica que houve uma falha durante o loop da vlan de controle.

Uso: Esse estado é mostrado nó Master.

*LINKS-UP (3)*

Significado: No estado LINKS-UP, todos os links relevantes foram verificados como operacionais e estão prontos para serem usados para a proteção de rede. Esse estado sugere que os links estão em funcionamento normal e podem participar da comutação de proteção.

Uso: Este estado é indicativo de que a rede está estável e os links estão ativos e funcionando corretamente. É mostrado nos nós de Transito.

*LINK-DOWN (4)*

Significado: O estado LINK-DOWN indica que um ou mais links monitorados pelo EAPS estão inativos ou falharam. Isso pode exigir que o protocolo inicie a comutação para um link de backup ou tome outras ações para manter a integridade da rede.

Uso: Esse estado sinaliza que há problemas com um link e o protocolo precisa tomar medidas para garantir que a rede permaneça operacional. É mostrado nos nós de Transito.

*PRE-FORWARDING (5)*

Significado: O estado PRE-FORWARDING é um estágio preparatório antes que um link comece a encaminhar tráfego normalmente. É um estado transitório onde o protocolo EAPS está preparando a rede para a comutação de proteção, mas o tráfego ainda não está sendo encaminhado.

Uso: Esse estado é usado para garantir que todas as verificações e preparações necessárias sejam realizadas antes que a comutação final ocorra.

Laboratório Prático

Topologia:

![](assets/img/posts/post-13/03.png)
 

Iremos configurar um cenário simples com EAPS, protegendo duas vlans: 100 e 200. A vlan de controle será a 4090 e os switches serão da própria Extreme Networks. (Mais detalhes sobre os switches expliquei neste artigo aqui). Infelizmente não conseguimos capturar os pacotes nesses SWs por limitação.

SW-1
``` 
# Module vlan configuration.
#
configure vlan default delete ports all
create vlan "EAPS-CONTROL"
configure vlan EAPS-CONTROL tag 4090
create vlan "v100"
configure vlan v100 tag 100
create vlan "v200"
configure vlan v200 tag 200
configure vlan EAPS-CONTROL add ports 1,3 tagged  
configure vlan v100 add ports 1,3-4 tagged  
configure vlan v200 add ports 1,3-4 tagged  

# Module eaps configuration.
#
configure eaps fast-convergence on
enable eaps
create eaps EAPS-DOMAIN
configure eaps EAPS-DOMAIN mode master
configure eaps EAPS-DOMAIN primary port 1
configure eaps EAPS-DOMAIN secondary port 3
enable eaps EAPS-DOMAIN
configure eaps EAPS-DOMAIN add protected vlan v100
configure eaps EAPS-DOMAIN add protected vlan v200
configure eaps EAPS-DOMAIN add control vlan EAPS-CONTROL
``` 
SW-2
``` 
# Module vlan configuration.
#
configure vlan default delete ports all
create vlan "EAPS-CONTROL"
configure vlan EAPS-CONTROL tag 4090
create vlan "v100"
configure vlan v100 tag 100
create vlan "v200"
configure vlan v200 tag 200
configure vlan EAPS-CONTROL add ports 1-2 tagged  
configure vlan v100 add ports 1-2 tagged  
configure vlan v200 add ports 1-2 tagged  

# Module eaps configuration.
#
configure eaps fast-convergence on
enable eaps
create eaps EAPS-DOMAIN
configure eaps EAPS-DOMAIN mode transit
configure eaps EAPS-DOMAIN primary port 1
configure eaps EAPS-DOMAIN secondary port 2
disable eaps EAPS-DOMAIN
configure eaps EAPS-DOMAIN add protected vlan v100
configure eaps EAPS-DOMAIN add protected vlan v200
configure eaps EAPS-DOMAIN add control vlan EAPS-CONTROL
``` 
SW-3

``` 
# Module vlan configuration.
#
configure vlan default delete ports all
create vlan "EAPS-CONTROL"
configure vlan EAPS-CONTROL tag 4090
create vlan "v100"
configure vlan v100 tag 100
create vlan "v200"
configure vlan v200 tag 200
configure vlan EAPS-CONTROL add ports 2-3 tagged  
configure vlan v100 add ports 2-4 tagged  
configure vlan v200 add ports 2-4 tagged  

# Module eaps configuration.
#
configure eaps fast-convergence on
enable eaps
create eaps EAPS-DOMAIN
configure eaps EAPS-DOMAIN mode transit
configure eaps EAPS-DOMAIN primary port 3
configure eaps EAPS-DOMAIN secondary port 2
enable eaps EAPS-DOMAIN
configure eaps EAPS-DOMAIN add protected vlan v100
configure eaps EAPS-DOMAIN add protected vlan v200
configure eaps EAPS-DOMAIN add control vlan EAPS-CONTROL
``` 
Dúvida: Os nós de transito precisam ter a porta primaria apontada para o master? Nos meus testes, não!

MK-RT-1
``` 
/interface vlan add interface=ether1 name=v100 vlan-id=100
/interface vlan add interface=ether1 name=v200 vlan-id=200
/ip address add address=10.1.0.1/30 interface=v100 network=10.1.0.0
/ip address add address=10.2.0.1/30 interface=v200 network=10.2.0.0
/system identity set name=MK-RT-1
``` 
MK-RT-2
``` 
/interface vlan add interface=ether1 name=v100 vlan-id=100
/interface vlan add interface=ether1 name=v200 vlan-id=200
/ip address add address=10.1.0.2/30 interface=v100 network=10.1.0.0
/ip address add address=10.2.0.2/30 interface=v200 network=10.2.0.0
/system identity set name=MK-RT-2
``` 
Verificando:

Comunicação MK1 x MK2

![](assets/img/posts/post-13/04.png)

Realizado um teste de ping entre os dois roteadores na rede da vlan 100, e comunicando normalmente.

SW-1

![](assets/img/posts/post-13/05.png)

No SW-1, que é o master, podemos ver o estado como Complete e que a porta 1 que é a primária está UP e a 3 que é a secundária está Blocked. Podemos ver também as vlans que estão sendo protegidas e a de controle (que está loopando) conforme vimos que é necessário para a proteção do EAPS.

![](assets/img/posts/post-13/06.png)
![](assets/img/posts/post-13/07.png)

Verificando a tabela MAC, constamos que recebemos MAC da porta 4 (SW-1 x MK1) e enviamos para a porta 1 (SW-1 x SW-2).

SW-2

![](assets/img/posts/post-13/08.png)

Podemos ver que o estado se encontra em Links-Up e recebemos MACs da porta 1 (SW-2 x SW-1) e enviamos para a porta 2 (SW-2 x SW-3).

SW-3

![](assets/img/posts/post-13/09.png)

Podemos ver que o estado se encontra em Links-Up e recebemos MACs da porta 2 (SW-2 x SW-3) e enviamos para a porta 4 (SW-3 x MK-2).

Validando a proteção do EAPS

Desabilitando a interface 1 do SW-2

![](assets/img/posts/post-13/10.png)
![](assets/img/posts/post-13/11.png)
![](assets/img/posts/post-13/12.png)

Explicando – A partir de 11:54

Recebimento de PDU de Flush de FDB: Indica que um anel de Ethernet foi detectado como inativo, e a tabela de endereços MAC deve ser atualizada ou limpa.

Transição de Estado do Domínio: Mostra que o estado do domínio EAPS mudou de Links-Up para Link-Down, sugerindo que houve uma falha.

Transição de Estado da Porta: Indica que a porta 1 foi detectada como inativa, mudando de Up para Down.

Verificando no SW-1

![](assets/img/posts/post-13/13.png)
![](assets/img/posts/post-13/14.png)

Explicando

Transição de Porta: A porta 3 (secondary) mudou de bloqueado para ativo, permitindo o encaminhamento de tráfego.

Transição de Estado do Domínio: O estado geral do domínio mudou de completo para falhado, indicando que houve um problema e a VLAN de controle deixou de loopar.

Recebimento de Link Down PDU: Um link foi detectado como inativo, e o endereço MAC especificado é o do link que falhou.

Verificando no SW-3

![](assets/img/posts/post-13/15.png)

Podemos ver que agora recebemos MACs da porta 3 (SW-1 x SW-3) e enviamos para a porta 4 (SW-3 x MK-2).

![](assets/img/posts/post-13/16.png)

Mesmas mensagens que recebemos anteriormente.

MK1

![](assets/img/posts/post-13/17.png)

Mesmo após a comutação, continuamos pingando normalmente sem perdas.

Dica Bônus:
Nos switches Datacom, é possível configurar EAPS sem precisar fazer isso em todos os dispositivos, utilizando MPLS. Veja como:

Infelizmente, não consegui reproduzir o processo em laboratório com as imagens do ExtremeXOS.

Agradecimentos ao mestre Leo Viana por essa dica valiosa.

A topologia abaixo ilustra uma rede real em produção.

![](assets/img/posts/post-13/18.png)

Estabelecemos um túnel L2VPN VPLS entre os dois modelos 4170 e entregamos a VLAN de controle ao DM4100 e as VLANS que serão protegidas. A configuração do EAPS foi necessária apenas neste último dispositivo. 

4170-1

![](assets/img/posts/post-13/19.png)

4170-2

![](assets/img/posts/post-13/20.png)

4100

![](assets/img/posts/post-13/21.png)


Referências
<https://en.wikipedia.org/wiki/Ethernet_Automatic_Protection_Switching>

<https://en.wikipedia.org/wiki/Ethernet_Ring_Protection_Switching>

<https://community.fs.com/article/ring-protection-protocol-erps-vs-eaps.html>

<https://datatracker.ietf.org/doc/rfc3619>

