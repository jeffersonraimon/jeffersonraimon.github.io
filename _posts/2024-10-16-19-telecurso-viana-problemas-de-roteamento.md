---
title: "19 | Telecurso Viana – Problemas de Roteamento"
date: 2024-10-16 00:00:00 -0300
categories: [Artigos]
tags: [Routing, Networking, Estudos]
image:
  path: assets/img/posts/post-19/capa.png
---

## Intro

Faço parte da equipe de CGR em um ISP e diariamente enfrentamos desafios que nos ajudam a consolidar o conhecimento. Meu gestor e mestre, Leo Viana, tem nos proporcionado diversas situações reais que estimulam nossa capacidade de resolução de problemas, o que chamamos de Telecurso Viana. Neste espaço, trarei algumas dessas questões, acompanhadas de explicações e demonstrações práticas em laboratório, para que possamos aprofundar nosso entendimento sobre os conceitos de roteamento e suas aplicações no dia a dia.

## Problema 01:

Um cliente está originando o prefixo ```100.64.1.0/24``` no Roteador 1 mas essa rede não está chegando no Roteador 4. Como resolver?

![](assets/img/posts/post-19/01.png)

Análise:

Podemos observar que os roteadores pertencem ao mesmo AS e a propagação é feita por BGP. Ou seja, iBGP.

Com isso matamos a charada. O iBGP não propaga rotas internas entre roteadores do mesmo sistema autônomo (AS) para evitar loops de roteamento.

Roteador 1:

![](assets/img/posts/post-19/02.png)

Roteador 2:

![](assets/img/posts/post-19/03.png)

Roteador 3:

![](assets/img/posts/post-19/04.png)

Roteador 4:

![](assets/img/posts/post-19/05.png)

Cada roteador aprende as rotas pelo iBGP das suas proprias sessões e não retransmite rotas aprendidas de outros iBGP.

Resoluções:

Para resolver podemos considerar as opções:

- Conectar e fechar um iBGP entre o Roteador 1 e 4
- BGP Full Mesh
- Criar um Route Reflector
- Utilizar BGP Confederation

A opçao 1 resolveria o problema, porém ainda assim o Roteador 2 e 4 / 1 e 3 não trocariam rotas caso fosse necessário futuramente, e teria que ser feito a mesma coisa, o que acabaria se tornando um Full Mesh (Opção 2).

A opção 2 é viavel apenas para poucos roteadores pois o problema do BGP full mesh é a escalabilidade. À medida que o número de roteadores aumenta, o número de conexões necessárias cresce rapidamente, o que resulta em:

- Número Excessivo de Conexões: Para n roteadores, são necessárias n(n−1)/2 conexões.
- Sobrecarga de Processamento: Múltiplas sessões de peering podem sobrecarregar os roteadores.
- Complexidade de Configuração: Gerenciar muitas conexões é complicado e propenso a erros.

Na opção 3 com Route Reflectors (RRs), o BGP reduz a necessidade de um full mesh da seguinte forma:

- Menos Conexões: Apenas os RRs se conectam a todos os roteadores, diminuindo o número total de peering.
- Propagação de Rotas: RRs recebem rotas dos clientes e as distribuem, eliminando a necessidade de conexões diretas entre todos os clientes.
- Maior Escalabilidade: A abordagem permite gerenciar redes maiores de forma mais eficiente.
- Menos Sobrecarga: Reduz a carga de processamento e memória nos roteadores.
Modificando a topologia original, utilizaremos o roteador 2 (por exemplo) como o Route Reflector

![](assets/img/posts/post-19/06.png)

```
router bgp 65001
 bgp router-id 10.2.2.2
 bgp cluster-id 10.2.2.2
 network 100.64.2.0 mask 255.255.255.0
 neighbor 10.1.1.1 remote-as 65001
 neighbor 10.1.1.1 update-source Loopback0
 neighbor 10.1.1.1 route-reflector-client
 neighbor 10.3.3.3 remote-as 65001
 neighbor 10.3.3.3 update-source Loopback0
 neighbor 10.3.3.3 route-reflector-client
 neighbor 10.4.4.4 remote-as 65001
 neighbor 10.4.4.4 update-source Loopback0
 neighbor 10.4.4.4 route-reflector-client
!
```
Para iBGP é recomendado o uso de IGP (nesse caso estou usando OSPF) para que todos os roteadores se alcancem e podermos utilizar os IPs de loopback deles (ou ter rotas para os IPs de enlace (/30)). Também importante definir o update-source com o IP de loopback do router. Se desejar pode utilizar o next-hop-self

Roteador 1:

![](assets/img/posts/post-19/07.png)

Roteador 4:

![](assets/img/posts/post-19/08.png)

A opção 4 é uma técnica utilizada para simplificar a administração de grandes sistemas autônomos (AS) BGP, o confederation permite que um único AS seja tratado como vários sub-AS, reduzindo a complexidade, minimizando o número de sessões BGP que precisam ser gerenciadas, já que os sub-AS comunicam-se entre si como se fossem parte de um único AS.

![](assets/img/posts/post-19/09.png)

AS Principal: 65001

Sub-AS: R1 – 64001 / R2 – 64002 / R3 -64003 – R4 – 64004

OBS:

Mantendo mesmo sub-AS nos roteadores não será repassado as rotas, por isso deve-se colocar sub-AS diferentes para cada roteador
Seperando os sub-AS, o peering sobe apenas com o /30 e não loopback (nos meus testes)
Roteador 1:
```
router bgp 64001
 bgp router-id 10.1.1.1
 bgp confederation identifier 65001
 bgp confederation peers 64002
 network 100.64.1.0 mask 255.255.255.0
 neighbor 10.12.0.2 remote-as 64002
```
![](assets/img/posts/post-19/10.png)

Roteador 2:
```
router bgp 64002
 bgp router-id 10.2.2.2
 bgp confederation identifier 65001
 bgp confederation peers 64001 64003 64004
 network 100.64.2.0 mask 255.255.255.0
 neighbor 10.4.4.4 remote-as 64004
 neighbor 10.12.0.1 remote-as 64001
 neighbor 10.23.0.2 remote-as 64003
 neighbor 10.24.0.2 remote-as 64004
```
![](assets/img/posts/post-19/11.png)

Roteador 3:
```
router bgp 64003
 bgp router-id 10.3.3.3
 bgp confederation identifier 65001
 bgp confederation peers 64002
 network 100.64.3.0 mask 255.255.255.0
 neighbor 10.23.0.1 remote-as 64002
```
![](assets/img/posts/post-19/12.png)

Roteador 4:
```
router bgp 64004
 bgp router-id 10.4.4.4
 bgp confederation identifier 65001
 bgp confederation peers 64000 64002
 network 100.64.4.0 mask 255.255.255.0
 neighbor 10.24.0.1 remote-as 64002
```
![](assets/img/posts/post-19/13.png)

## Problema 02:

Um cliente não recebe as rotas do Roteador 1 e 2 no Roteador 4 e vice-versa, pelo peering com o AS 65206 (foi validado que o AS permite a passagem das rotas). Como resolver?

![](assets/img/posts/post-19/14.png)

Análise:

O BGP, por padrão, rejeita rotas que contêm o próprio ASN na lista de ASNs para evitar loops de roteamento. Quando um roteador recebe uma rota com seu próprio ASN, isso indica que a rota pode estar voltando para ele, o que poderia levar a situações em que os pacotes ficam circulando indefinidamente entre roteadores.

Resolução:

O ```allowas-in``` é uma configuração utilizada em protocolos de roteamento BGP (Border Gateway Protocol) para permitir que um roteador aceite rotas que contêm o seu próprio ASN (Autonomous System Number) na lista de ASNs.

Roteador 2:
```
router bgp 65001
 bgp log-neighbor-changes
 network 100.64.2.0 mask 255.255.255.0
 neighbor 10.12.0.1 remote-as 65001
 neighbor 10.23.0.2 remote-as 65206
 neighbor 10.23.0.2 allowas-in
```
![](assets/img/posts/post-19/15.png)

Roteador 4:
```
router bgp 65001
 bgp log-neighbor-changes
 network 100.64.4.0 mask 255.255.255.0
 neighbor 10.34.0.1 remote-as 65206
 neighbor 10.34.0.1 allowas-in
```
![](assets/img/posts/post-19/16.png)


## Problema 03:

Um cliente questionou se seria possível subir a sessão BGP com nossos 2 roteadores utilizando um único IP de enlace. O cenário seria o seguinte:

![](assets/img/posts/post-19/17.png)

Análise:

Podemos ver que o cliente ira fazer um ponto a ponto com um switch L3,e deverá ter conectividade com as loopbacks dos 2 roteadores para fechar a sessão BGP. Porém também temos o problema do prefixo do cliente por dentro da rede interna, pois os switches devem saber o caminho da volta até o roteador do cliente.

Resoluções:

O roteador do cliente não está diretamente conectado, e para isso deveremos usar o ebgp multi-hop. E para o roteamento do prefixo do cliente nos dois switches podemos usar:

- MPLS
- Redistribute

Solução com redistribute para switches que não possuem MPLS

![](assets/img/posts/post-19/18.png)

O roteador R1/R2 irá fechar a sessão BGP via loopback e receber o prefixo do cliente via BGP e irá originar rota default para dentro do OSPF.
```
R1#show run | s bgp
router bgp 65001
 bgp log-neighbor-changes
 neighbor 10.23.0.1 remote-as 1000
 neighbor 10.254.100.2 remote-as 65206
 neighbor 10.254.100.2 ebgp-multihop 3
 neighbor 10.254.100.2 update-source Loopback0

R1#show run | s ospf
router ospf 1
 router-id 172.16.100.1
 passive-interface Loopback0
 default-information originate always
```
Para evitar loop de roteamento, no SW5 devemos criar uma rota estática do prefixo do cliente e realizar o redistribute no OSPF (podemos utilizar filtro para restringir apenas esse prefixo).

![](assets/img/posts/post-19/19.png)

SW5
```
router ospf 1
 router-id 10.5.5.5
 redistribute static subnets route-map AS65206-ROUTE
 passive-interface Loopback0
 passive-interface Vlan100 // /30 com o cliente

ip route 100.70.6.0 255.255.255.0 10.254.100.2
ip prefix-list AS65206 seq 5 permit 100.70.6.0/24

route-map AS65206-ROUTE permit 10
 match ip address prefix-list AS65206
```
![](assets/img/posts/post-19/20.png)

Foi adicionado outro roteador para fazer o papel de upstream para realizar testes.

![](assets/img/posts/post-19/21.png)

![](assets/img/posts/post-19/22.png)


![](assets/img/posts/post-19/23.png)

Solução com MPLS (BGP FREE CORE)

Solução mais fácil porém depende dos switches terem suporte ao MPLS. Com o MPLS aplicado nos RTs e SWs, o switch 4 não precisa verificar as rotas e sim apenas os labels, com isso encaminhará os frames normalmente para o SW5. O SW5 deverá ter a rota estática do prefixo do cliente configurada para saber para quem entregar. Também tivamos a recursividade para LSP. Exemplo em huawei: route recursive-lookup tunnel

![](assets/img/posts/post-19/24.png)

![](assets/img/posts/post-19/25.png)

Fim!