---
title: "11 | Migrando para IPv6 com a ajuda do NTPv6"
date: 2024-08-16 00:00:00 -0300
categories: [Tutoriais]
tags: [IPv6, NTPv6, Networking]
image:
  path: assets/img/posts/post-11/capa.png
---

Eu tenho um homelab em casa e estou atualmente migrando minhas VMs para IPv6. Durante esse processo, enfrentei um desafio e irei abordar como resolvi de forma satisfatória, creio que possa ajudar mais alguém.

O problema:

Configurei minhas VMs com endereços IPv6 ULA (Unique Local Address) de forma estática para facilitar o gerenciamento. No entanto, desejo que algumas delas também tenham acesso à Internet por IPv6. Como os endereços ULA são estáticos e meu provedor de Internet me fornece endereços IPv6 GUA (Global Unicast Address) de forma dinâmica, sempre que ocorre uma mudança no prefixo fornecido pelo provedor, eu teria que ajustar manualmente as configurações das minhas VMs. Felizmente, existe um método que tem sido bastante útil para lidar com essa situação: o NPTv6 (Network Prefix Translation para IPv6).

O NTPv6 de forma resumida:

O NPTv6 oferece uma forma de traduzir de maneira STATELESS entre prefixos ULA e GUA do mesmo tamanho. Como o NPTv6 é stateless, não é necessária a tradução de portas nem outra manipulação das características da camada de transporte (OSI), com isso é mantido a filosofia da comunicação fim-a-fim proporcionada pelo IPv6 (o que era quebrado com o NAT do IPv4).

![](assets/img/posts/post-11/01.png)

A Simple [NPTv6] Translator” (courtesy of IETF RFC 6296)
Nas minhas pesquisas vi também que é uma opção para utilizar em cenários multihoming sem a necessidade de ser um AS.

Agora vamos ao processo da implementação do NTPv6 na minha rede

Topologia resumida:

![](assets/img/posts/post-11/capa.png)

1: Recebo um /56 do provedor por meio de DHCPv6-Client

![](assets/img/posts/post-11/02.png)

2: Quebro esse GUA /56 em pools /64

![](assets/img/posts/post-11/03.png)

Adiciono os IPS na interface (sem EUI64 mesmo e advertise). o GUA é a partir da pool “ipv6” e o ULA de forma estática.

![](assets/img/posts/post-11/04.png)

3: Adicionado ULA de forma estática (o ::55 seria por conta do VRRP que rodo com outra maquina).

![](assets/img/posts/post-11/05.png)
 

NTPv6 no RouterOSv7

No RouterOSv7 (se não me engano na versão 7.1) foi introduzido a opção de NTPv6 no menu de Firewall (Mangle).

Para configurar vamos precisar de 2 regras Mangle:

1 – chain Postrouting com action snpt
```
/ipv6 firewall mangle add action=snpt chain=postrouting comment=SNPT-s dst-prefix=2001:db8:4846:d801::/64 out-interface=LINK src-address=fd10:1:10::/64 src-prefix=fd10:1:10::/64
```
![](assets/img/posts/post-11/06.png)
![](assets/img/posts/post-11/07.png)

Explicação:

- ```/ipv6 firewall mangle add```: Adiciona uma nova regra ao firewall para manipulação de pacotes IPv6.
- ```action=snpt```: Define a ação como “source network prefix translation” (SNPT). Isso significa que a regra está realizando uma tradução do prefixo de rede de origem dos pacotes.
- ```chain=postrouting```: Especifica que esta regra deve ser aplicada após o roteamento dos pacotes. Isso é típico para manipulações que precisam ocorrer quando os pacotes estão saindo da interface.
- ```comment=SNPT-s```: será útil para o script que irei explicar mais pra frente
- ```dst-prefix=2001:db8:4846:d801::/64```: Define o prefixo de destino para o qual a tradução será aplicada. Ou seja, pacotes que têm um destino dentro deste prefixo serão afetados pela regra.
- ```out-interface=LINK```: Aplica a regra apenas aos pacotes que estão saindo pela interface LINK
- ```src-address=fd10:1:10::/64```: Especifica o prefixo de origem dos pacotes que serão traduzidos.
- ```src-prefix=fd10:1:10::/64```: Define o prefixo de origem que será traduzido. Neste caso, o prefixo de origem especificado é igual ao prefixo de origem dos pacotes, indicando que a tradução será feita para esse prefixo.

2 – chain Prerouting com action dnpt
```
/ipv6 firewall mangle add action=dnpt chain=prerouting comment=DNPT-s dst-address=2001:db8:4846:d801::/64 dst-prefix=fd10:1:10::/64 in-interface=LINK src-prefix=2001:db8:4846:d801::/64
```
![](assets/img/posts/post-11/08.png)
![](assets/img/posts/post-11/09.png)

Explicação:

- ```/ipv6 firewall mangle add```: Adiciona uma nova regra ao firewall para manipulação de pacotes IPv6.
- ```action=dnpt```: Define a ação como “destination network prefix translation” (DNPT). Isso significa que a regra está realizando uma tradução do prefixo de rede de destino dos pacotes.
- ```chain=prerouting```: Especifica que esta regra deve ser aplicada antes do roteamento dos pacotes, ou seja, enquanto os pacotes estão chegando na interface.
- ```comment=DNPT-s```: será útil para o script que irei explicar mais pra frente
- ```dst-address=2804:88c:4846:d801::/64```: Define o prefixo de destino para o qual a tradução será aplicada. Ou seja, pacotes que têm um destino dentro deste prefixo serão afetados pela regra.
- ```dst-prefix=fd10:1:10::/64```: Define o prefixo de destino que será traduzido para o prefixo especificado.
- ```in-interface=LINK-VOANET-500MB```: Aplica a regra apenas aos pacotes que estão chegando pela interface LINK.
- ```src-prefix=2804:88c:4846:d801::/64```: Define o prefixo de origem que será considerado para a tradução de destino.

OBS: Estou especificando a interface pois nos meus testes, sem setar a interface, eu não conseguia pingar o bloco ```fd10:1:10::/64``` de forma interna

Validando internamente:

Podemos ver que consigo pingar normalmente o google.com.br em IPv6, mesmo essa VM possuindo apenas IPv6 ULA

![](assets/img/posts/post-11/10.png)

Infelizmente, o traceroute deixa de funcionar corretamente (não fui a fundo para entender porque).

![](assets/img/posts/post-11/11.png)

Porém, analisando com o tcpdump, podemos ver que o processo do traceroute está ocorrendo normalmente.

![](assets/img/posts/post-11/12.png)

Validando na RB

![](assets/img/posts/post-11/13.png)

Validando externamente:
ICMPv6

Eu deixo o ICMPv6 liberado como explico no post 06 

![](assets/img/posts/post-11/14.png)

OBS: Tenho um domínio e com o type DNS AAAA apontado para o IPv6 que é feito a tradução do NTPv6 nessa VM, mesmo recebendo IPv6 GUA dinâmico. Explico como fiz aqui

Podemos usar o tcpdump e validar o teste acima


![](assets/img/posts/post-11/15.png)

TCP/UDP

Como tenho alguns serviços rodando nas VMs como um servidor de RustDesk, preciso que fique aberto essas portas externamente. Nessa maquina estou rodando um Traefik Proxy e expondo as portas TCP/UDP necessárias. Vamos validar que a comunicação fim-a-fim se mantém integra.

Estou exportando as seguintes portas

![](assets/img/posts/post-11/16.png)
![](assets/img/posts/post-11/17.png)

Verificando com o tcpdump

![](assets/img/posts/post-11/18.png)
 

Como podemos validar, mesmo como NTPv6, a VM fica exposta, e com isso é importante trabalhar com um firewall diretamente na VM (Ex. ip6tables ou UFW) e controlar as portas.

Automatizando
Tenho um script que sempre que o GUA é alterado, é ajustado nas regras Mangle.

Basicamente eu tenho configurado um Scheduler na RB que de 24h em 24h executa o script

Você pode encontrar [aqui.](https://github.com/jeffersonraimon/scripts-mk/tree/main)