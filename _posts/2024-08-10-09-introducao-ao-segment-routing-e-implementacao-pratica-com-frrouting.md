---
title: "09 | Introdução ao Segment Routing e Implementação Prática com FFRouting"
date: 2024-08-10 00:00:00 -0300
categories: [Artigos]
tags: [SegmentRouting, FRRouting, MPLS]
image:
  path: assets/img/posts/post-09/capa.png
---

No mundo das redes, a necessidade de simplificação e eficiência sempre foi uma prioridade. O Segment Routing (SR) surge como uma solução inovadora na engenharia de tráfego e gerenciamento de redes. Vamos explorar o problema que o Segment Routing resolve, os conceitos-chave associados, e fornecer uma visão geral de um laboratório básico que será realizado usando FFRouting.

O problema que o Segment Routing veio resolver

Tradicionalmente, o roteamento de pacotes em redes IP e MPLS envolvia a manutenção de tabelas de roteamento complexas e a necessidade de protocolos de sinalização para estabelecer caminhos, como o LDP e RSVP. Isso poderia levar a uma gestão e configuração de rede complexas, além de aumentar o overhead de processamento e memória nos roteadores. Além disso, a falta de flexibilidade e a necessidade de protocolos de sinalização pesados, como o RSVP-TE (Resource Reservation Protocol – Traffic Engineering), criavam desafios adicionais na engenharia de tráfego.

O Segment Routing foi desenvolvido para resolver esses problemas ao simplificar a engenharia de tráfego e reduzir a complexidade da rede. Com o SR, a rede pode ser configurada de forma mais eficiente, e os pacotes podem ser roteados de maneira mais direta e flexível.

Conceitos Básicos do Segment Routing

Para entender o Segment Routing, vamos refrescar alguns conceitos:

- Labels: Um label é uma identificação que é usada para direcionar pacotes ao longo de um caminho específico na rede (LSP – Label Switched Path). Em vez de manter uma tabela de roteamento complexa, os roteadores usam labels para tomar decisões rápidas sobre como encaminhar pacotes. Simulando assim um circuito virtual. Bastante util por exemplo em redes BGP Free Core para evitar ter que rodar BGP em Switches L3/Roteadores no meio do caminho.

![](assets/img/posts/post-09/01.png)

-  SR-MPLS (Segment Routing com MPLS): O SR-MPLS é uma aplicação do Segment Routing sobre MPLS (Multiprotocol Label Switching). Em vez de usar protocolos tradicionais de sinalização para definir caminhos, o SR-MPLS utiliza labels de segmentação para especificar a rota do pacote de forma mais direta e eficiente. Utiliza o MPLS como data plane.

-  SRv6 (Segment Routing com IPv6): O SRv6 é uma implementação do Segment Routing sobre IPv6. Em vez de usar labels MPLS, o SRv6 usa endereços IPv6 como segmentos para direcionar pacotes. Isso permite maior flexibilidade e escalabilidade, aproveitando as capacidades avançadas do IPv6. Descartando assim a necessidade do MPLS (por isso dizem que o SR irá acabar com o MPLS).

O Segment Routing é configurado diretamente no IGP (OSPF ou IS-IS) não necessitando de outros protocolos. Utiliza o paradigma de roteamento baseado na origem, ou seja, podemos definir todo o caminho dos pacotes com engenharia de trafego já na origem.

Laboratório Básico com FFRouting e SR-MPLS
Para ilustrar a implementação prática do Segment Routing, será realizado um laboratório básico utilizando FFRouting e SR-MPLS. Futuramente farei outro laboratório com SRv6.

Topologia:

![](assets/img/posts/post-09/02.png)
 

- Temos 6 GNU/Linux Ubuntu 20.04 LTS com FFRouting 10.1 com configuração mínima de enlaces /30 e OSPF
- Rede A: 172.16.0.0/24
- Rede B: 172.18.0.0/24

Com essa configuração minima, vamos verificar o caminho de 172.16.0.1 (Linux-SRV-18) até 172.18.0.1 (Linux-SRV-19)

![](assets/img/posts/post-09/03.png)

Para validar melhor, subi um servidor zabbix e desenhei o mapa da topologia e vamos acompanhar por onde o tráfego está passando. Irei utilizar o iperf3 para gerar tráfego entre o SRV-18 e SRV-19.

![](assets/img/posts/post-09/04.png)

Podemos ver que pela decisão do OSPF, o tráfego está indo pelo caminho de cima.

Agora vamos configurar o SR para brincar um pouco.

Configuração do Segment Routing no OSPF e SR-MPLS

Irei mostrar apenas do R1, basicamente basta repetir nos outros roteadores com os ajustes de IPs.
``` 
router ospf
 ospf router-id 10.1.1.1
 capability opaque
 segment-routing on
 segment-routing node-msd 8
 segment-routing prefix 10.1.1.1/32 index 1
 router-info area
exit
!
``` 

- ``` capability opaque``` : Os LSAs opacos têm a finalidade de fornecer uma maneira flexível de transportar informações que não estão diretamente relacionadas ao protocolo OSPF em si, mas são úteis para aplicações adicionais ou funcionalidades extendidas. O Segment Routing utiliza informações adicionais que não são cobertas pelos LSAs padrão do OSPF, como os Segment Identifiers (SIDs) usados no Segment Routing.

- ``` segment-routing node-msd 8``` : define um valor específico para o Maximum Segment Depth (MSD) para um nó em uma rede MPLS. Em outras palavras, MSD especifica o limite máximo de segmentos (ou etiquetas) que um roteador pode empilhar e manipular ao encaminhar pacotes. Esse valor é importante para garantir que o tráfego que passa pelo roteador não exceda a capacidade que o roteador pode gerenciar. O FFR aceita o valor de 1 a 16.

- ``` segment-routing prefix 10.1.1.1/32``` : Serve para associar um Segment Identifier (SID) a um prefixo IP específico. Geralmente utilizamos o IP de Loopback.

- ``` index 1``` : O SR utiliza um range global de labels de 16000 a 23999 (SRGB). Com o index 1, informa que esse roteador terá o label 16001 para o SID. Com o index 2, seria 16002 por exemplo.

- ``` router-info area``` : usado em configurações de OSPF para garantir que o roteador anuncie suas capacidades e informações relevantes para outros roteadores na mesma área OSPF. Será util para transmitir informações referente ao SR.

Verificando Segment Routing Database:
``` 
show ip ospf database segment-routing self-originate 
show ip ospf database segment-routing //para ver de todos os outros SR-Nodes
``` 
![](assets/img/posts/post-09/05.png)

Verificando os Labels do MPLS
``` 
show mpls table
``` 
![](assets/img/posts/post-09/06.png)

Após a conclusão das configurações de OSPF e SR-MPLS, o SR já está em funcionamento no seu modo SR-BE (best-effort).

“O SR-BE é o modo mais puro do SR-MPLS, em que o OSPF calcula a rota mais curta para o pacote seguir” [Huawei 2023].

Criação de Caminhos de Engenharia de Tráfego

Definir os caminhos que os pacotes devem seguir usando os labels configurados. Isso inclui a configuração de políticas de tráfego e regras de encaminhamento.

Vamos criar uma engenharia de trafego fazendo com que o (Linux-SRV-18) 172.16.0.1 se comunique com (Linux-SRV-19) 172.18.0.1 pela rota de baixo.
``` 
segment-routing
 traffic-eng
  segment-list SL1
   index 10 mpls label 16002
   index 20 mpls label 16004
   index 30 mpls label 16005
   index 40 mpls label 16003
   index 50 mpls label 16006
  exit
  ``` 
Criamos um segment-list para definir a engenharia de tráfego. Como foi definido o valor de index igual ao numero do roteador na topologia, cada um ficou com o label correspondente. Exemplo: RT5 = 16005.

``` 
policy color 10 endpoint 10.6.6.6
   name ROTA-DE-BAIXO
   binding-sid 1111
   candidate-path preference 100 name ROTA-DE-BAIXO explicit segment-list SL1
  exit
  ``` 
- ``` policy color``` : Define uma política de Segment Routing associada à cor 10. Em Segment Routing, cores são usadas para identificar políticas de engenharia de tráfego.

- ``` endpoint``` : Especifica o endpoint para o qual a política de Segment Routing é aplicada. Neste caso, o endpoint é o endereço de loopback do RT6.

- ``` binding-sid``` : Associa um Segment Identifier (SID) ao endpoint definido. Esse valor podemos definir para associar a essa Policy.

- ``` candidate-path``` : Define um caminho candidato que pode ser utilizado para encaminhamento. Onde podemos definir preferencias e qual segment-list irá ser utilizado.

Verificando
``` 
show mpls table
show sr-te policy
show sr-te policy detail
``` 
![](assets/img/posts/post-09/07.png)

É possível observar os labels que o pacote deve passar ao longo do caminho

Teste e Validação

Para testar de forma rápida, no R1 irei criar um rota estática e definindo como gateway o ip 10.6.6.6, que é o endpoint do SR-TE e colocar o color 10. Com isso a policy entrará em ação.
``` 
ip route 172.18.0.0/24 10.6.6.6 color 10
``` 
![](assets/img/posts/post-09/08.png)

Testando com traceroute no SRV-18

![](assets/img/posts/post-09/09.png)

Podemos ver que houve mudança no salto 2, porém não tivemos resposta ICMP. Vamos validar pelo zabbix por onde o tráfego está indo.

![](assets/img/posts/post-09/10.png)

Pelo zabbix podemos validar por onde o tráfego está passando.

Capturando pacotes no R1 podemos ver toda a pilha de labels do caminho.

![](assets/img/posts/post-09/11.png)
 

Conclusão

O Segment Routing representa um avanço significativo na engenharia de tráfego e na gestão de redes, oferecendo uma abordagem mais simplificada e eficiente em comparação com métodos tradicionais.

Referências

[IETF. (2020). Segment Routing with MPLS Data Plane. RFC 8402.](https://datatracker.ietf.org/doc/html/rfc8402)

[IETF. (2021). Segment Routing over IPv6 (SRv6). RFC 8986.](https://datatracker.ietf.org/doc/html/rfc8986)

[Instalação do FRRouting (FRR) – Roteamento dinâmico no seu linux +Bônus iBGP](https://blog.remontti.com.br/4771)

[FFR - Segment Routing](https://docs.frrouting.org/en/latest/ospfd.html#segment-routing)

[FFR - PATH](https://docs.frrouting.org/en/latest/pathd.html)

[SEGMENT ROUTING: ROTEAMENTO DE TRÁFEGO BASEADO NA ORIGEM](https://repositorio.ifpb.edu.br/bitstream/177683/3549/1/Jos%C3%A9%20Silvestre%20da%20Silva%20Galv%C3%A3o%20-%20TCC.pdf)