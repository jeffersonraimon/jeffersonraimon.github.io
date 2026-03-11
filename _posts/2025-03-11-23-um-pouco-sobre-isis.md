---
title: "23 | Um pouco sobre IS-IS"
date: 2025-03-11 00:00:00 -0300
categories: [Artigos]
tags: [ISIS, Routing, IGP, Networking]
image:
  path: assets/img/posts/post-23/capa.png
---

O IS-IS (Intermediate System to Intermediate System) é um protocolo de roteamento utilizado em redes de computadores para determinar o melhor caminho entre os dispositivos de rede, como roteadores. Ele é um protocolo de roteamento de link-state (estado de enlace), o que significa que ele constrói e mantém uma visão completa da topologia da rede, trocando informações de estado de enlace entre os dispositivos para calcular os melhores caminhos. O IS-IS é comumente usado em redes de grande escala, como provedores de serviços de internet (ISPs) e data centers, devido à sua escalabilidade e robustez.

O que significa IS-IS?

Intermediate System = Roteador

End System = Host final

![](assets/img/posts/post-23/01.png)

Como funciona o IS-IS?

O IS-IS utiliza uma abordagem de roteamento baseada em áreas e níveis. Ele funciona de forma semelhante ao OSPF (Open Shortest Path First), mas com algumas diferenças importantes em sua implementação. O IS-IS distribui informações de estado de enlace, onde cada roteador mantém uma tabela que descreve a topologia da rede. A partir dessas informações, o roteador pode calcular os melhores caminhos para os pacotes de dados com base no algoritmo SPF (Shortest Path First), que considera os custos dos enlaces.

Significado do NET (Network Entity Title)

No IS-IS, cada roteador é identificado por um título único chamado NET (Network Entity Title). O NET é utilizado para identificar de forma única cada roteador na rede IS-IS e é similar ao conceito de endereço IP, mas no contexto do IS-IS. O NET é formado por um identificador que combina a rede administrativa com um número exclusivo, o que facilita a distinção entre os roteadores em diferentes redes.

NET consiste em Area, System ID e NSEL. A própria Area consiste em AFI (Address Family Identifier) ​​e Area ID.

A Area pode ter comprimento variável de 1 a 13 bytes, o System ID é de 6 bytes e o NSEL – 1 byte.

Vamos verificar um exemplo de NET de 49.001.1111.1111.1111.00. 

- 49 é o AFI, e no caso de 49 significa “espaço de endereço privado”, semelhante ao RFC1918 para IPv4.
- 0001 é a Area ID
- 49.0100 é a Area
- 1111.1111.1111 é o System ID (semelhante ao Router-ID)
- 00 é NSEL, que deve ser zero. Se não for zero, então nenhuma adjacência IS-IS é formada.

Levels no IS-IS

O IS-IS trabalha com dois níveis de roteamento, conhecidos como Level 1 e Level 2. Essa abordagem de níveis facilita a escalabilidade em grandes redes, permitindo a segmentação e o gerenciamento da rede de forma eficiente.

- Level 1: O roteamento de Level 1 acontece dentro de uma área específica. Roteadores que pertencem à mesma área comunicam-se diretamente entre si e são responsáveis por rotear pacotes dentro dessa área. O IS-IS utiliza o conceito de “áreas” para organizar a rede.
- Level 2: O roteamento de Level 2 ocorre entre áreas diferentes. Roteadores de Level 2 são responsáveis por interconectar áreas diferentes e realizar o roteamento entre elas. Eles mantêm informações de topologia de toda a rede e têm uma visão global da estrutura de roteamento.
Ou seja, roteadores configurados como Level 1 apenas possuem rotas da sua área, a não ser que seja feito uma redistribuição (veremos mais adiante).

Como os roteadores se comunicam?
Para que os roteadores IS-IS se comuniquem e troquem informações de estado de enlace, eles utilizam pacotes de informações chamados CSNPs (Complete Sequence Number PDU) e PSNPs (Partial Sequence Number PDU). Esses pacotes são usados para garantir que todos os roteadores tenham uma visão consistente da topologia da rede. Podemos notar que essas trocas de informações é feito em camada 2.

- Hello PDU: Quando um roteador IS-IS se inicia, ele envia pacotes Hello PDU para identificar outros roteadores na mesma rede e formar adjacências. Esses pacotes contêm informações sobre o NET e outros parâmetros necessários para a comunicação.

![](assets/img/posts/post-23/02.png)

- Link-State PDU (LSP): Depois que as adjacências são formadas, os roteadores começam a trocar informações de estado de enlace por meio de LSPs. Cada LSP contém informações sobre o estado de um link, como o custo e a disponibilidade de um enlace.

![](assets/img/posts/post-23/03.png)

- LSP Flooding: Os LSPs são propagados pela rede através de flooding (disseminação). Quando um roteador recebe um LSP, ele verifica se o LSP é novo ou se já foi recebido anteriormente, garantindo que a rede tenha uma visão consistente e atualizada.
Cálculo de Rota: Com as informações de LSP recebidas, cada roteador recalcula a melhor rota para alcançar destinos dentro da rede, utilizando o algoritmo SPF para determinar o caminho mais curto.

Exemplo prático

Topologia:

Foi utilizado Roteadores Cisco CSR 1000V

![](assets/img/posts/post-23/capa.png)


RT1
```
interface Loopback0
 ip address 10.1.1.1 255.255.255.255
!
interface GigabitEthernet1
 ip address 10.0.12.1 255.255.255.252
 ip router isis 1
!
interface GigabitEthernet4
 ip address 172.16.10.254 255.255.255.0
!
router isis 1
 net 49.0001.1111.1111.1111.00
 is-type level-1
 passive-interface GigabitEthernet4
 passive-interface Loopback0
!
```
RT2
```
interface Loopback0
 ip address 10.2.2.2 255.255.255.255
!
interface GigabitEthernet1
 ip address 10.0.12.2 255.255.255.252
 ip router isis 1
!
interface GigabitEthernet2
 ip address 10.0.23.1 255.255.255.252
 ip router isis 1
!
interface GigabitEthernet4
 ip address 172.16.20.254 255.255.255.0
!
router isis 1
 net 49.0001.2222.2222.2222.00
 passive-interface GigabitEthernet4
 passive-interface Loopback0
!
```
RT3
```
interface Loopback0
 ip address 10.3.3.3 255.255.255.255
!
interface GigabitEthernet1
 ip address 10.0.23.2 255.255.255.252
!
interface GigabitEthernet4
 ip address 172.16.30.254 255.255.255.0
!
router isis 1
 net 49.0002.3333.3333.3333.00
 passive-interface GigabitEthernet4
 passive-interface Loopback0
!
```

Podemos ver que sao fechados adjacências separadas entre L1 e L2

![](assets/img/posts/post-23/05.png)

Verificando as Tabelas de Roteamento
RT1

![](assets/img/posts/post-23/06.png)


Podemos notar que não temos rotas para as redes de RT3 por pertencer a outra área. Porém recebemos automaticamente rota default via RT2.

RT2

![](assets/img/posts/post-23/07.png)


RT3

![](assets/img/posts/post-23/08.png)

No RT3 temos rotas automaticamente até para o RT1. Isso aconteceu porque o RT2 faz esse intermédio por ser L1/L2.

Como enviar as rotas de RT3 para RT1?
Para isso no RT2 faremos redistribuição de ISIS Level 2 em Level 1

```
router isis 1
 redistribute isis ip level-2 into level-1 route-map RT3-LAN
!
ip prefix-list RT3-LAN seq 5 permit 172.16.30.0/24 le 32
ip prefix-list RT3-LAN seq 10 permit 10.3.3.3/32
!
route-map RT3-LAN permit 10 
 match ip address prefix-list RT3-LAN
!
```
![](assets/img/posts/post-23/09.png)

MPLS com ISIS
```
router isis 1
 mpls ldp sync
 mpls ldp autoconfig
!
mpls ldp router-id Loopback0
!
mpls ip
!
```

![](assets/img/posts/post-23/10.png)

Conclusão

O IS-IS é um protocolo de roteamento robusto e eficiente, utilizado principalmente em redes grandes e complexas. Ele utiliza conceitos de níveis (Level 1 e Level 2) para fornecer escalabilidade e organização da rede, além de usar o NET como identificador único dos roteadores. A comunicação entre os roteadores é feita através de pacotes específicos, como Hello PDU, LSPs e CSNPs, garantindo que todos os roteadores tenham uma visão sincronizada da rede. Por sua eficiência e flexibilidade, o IS-IS é uma escolha popular para grandes implementações de rede, como em provedores de serviços de internet e grandes data centers.

Referências

[IS-IS](https://en.wikipedia.org/wiki/IS-IS)

