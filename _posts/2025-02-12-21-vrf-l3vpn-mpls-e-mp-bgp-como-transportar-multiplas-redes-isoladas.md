---
title: "21 | VRF, L3VPN, MPLS e MP-BGP: Como transportar múltiplas redes isoladas"
date: 2025-02-12 00:00:00 -0300
categories: [Artigos]
tags: [VRF, MPLS, MPBGP, L3VPN, Networking]
image:
  path: assets/img/posts/post-21/capa.png
---

Já faz um tempo que venho estudando sobre VRF, L3VPN, MP-BGP e VPNv4, e sempre achei esses conceitos muito interessantes. No começo, pode parecer um monte de siglas complicadas, mas quando a gente entende como elas se conectam, tudo começa a fazer sentido.

Neste post, vou explicar de forma simples como essas tecnologias funcionam e se relacionam, e, claro, vou montar um lab prático para demonstrar tudo isso na prática. 

![](assets/img/posts/post-21/capa.png)

Cenário do Laboratório

No meu lab, eu sou o AS65010, operando um backbone entre Rio de Janeiro (RJ) e Bahia (BA). Esse backbone utiliza OSPF e MPLS para garantir conectividade entre os sites.

Os roteadores que conectam os clientes também rodam BGP e fazem troca de rotas via iBGP com VPNv4, permitindo o transporte de múltiplas VPNs sobre a mesma infraestrutura.

## Clientes do Ambiente

---

### Cliente A

| Propriedade | Valor |
|-------------|------|
| AS | 65020 |
| Localidades | RJ e BA |

### Prefixos utilizados

| Localidade | Prefixo |
|-----------|--------|
| RJ | 172.16.1.0/24 |
| BA | 172.16.2.0/24 |

### Conectividade

- Em cada localidade é estabelecido **eBGP** com o cliente.

---

### Cliente B

| Propriedade | Valor |
|-------------|------|
| AS | Não possui AS próprio |
| Protocolo interno | OSPF |

### Prefixos utilizados

| Localidade | Prefixo |
|-----------|--------|
| RJ | 172.16.1.0/24 |
| BA | 172.16.2.0/24 |

### Transporte de rotas

- Como o cliente **não roda BGP**, as rotas são **redistribuídas via OSPF dentro da VRF**.
- Isso garante o **transporte dos prefixos entre RJ e BA**.

---

## Roteadores de Borda

### Borda 1 — RT3

| Propriedade | Valor |
|-------------|------|
| Equipamento | Cisco CSR1000v |
| Sistema | IOS XE |

---

### Borda 2 — RT7

| Propriedade | Valor |
|-------------|------|
| Equipamento | Huawei AR1000V |

O objetivo desse lab é demonstrar na prática como essas tecnologias se integram para isolar e transportar múltiplas redes sobre um backbone MPLS usando VRF, MP-BGP e VPNv4.

Nesse post não vou abordar as configurações do Backbone MPLS e IBGP, e IPv6 para focar nos tópicos principais.

## VRF

Após o backbone finalizado, no RT3 e RT7 irei criar uma VRF para o cliente A e outra para o B.

VRF (Virtual Routing and Forwarding) é uma tecnologia que permite criar várias tabelas de roteamento separadas no mesmo roteador.

- Funciona como “roteadores virtuais” dentro de um único equipamento.
- Cada VRF tem sua própria tabela de roteamento, isolada das outras.
- Permite separar redes de diferentes clientes ou serviços sem precisar de múltiplos roteadores.

RT3 – CISCO
```
vrf definition CLIENTE-A
 rd 100:1
 !
 address-family ipv4
  route-target export 100:1
  route-target import 100:1
 exit-address-family
!
vrf definition CLIENTE-B
 rd 200:1
 !
 address-family ipv4
  route-target export 200:1
  route-target import 200:1
 exit-address-family
!
interface GigabitEthernet1
 vrf forwarding CLIENTE-A
 ip address 10.1.13.3 255.255.255.240
 ```

RT7 – HUAWEI – VRF é VPN-INSTANCE
```
ip vpn-instance CLIENTE-A
 ipv4-family
  route-distinguisher 100:1
  vpn-target 100:1 export-extcommunity
  vpn-target 100:1 import-extcommunity
#
ip vpn-instance CLIENTE-B
 ipv4-family
  route-distinguisher 200:1
  vpn-target 200:1 export-extcommunity
  vpn-target 200:1 import-extcommunity
#
interface GigabitEthernet0/0/3
 ip binding vpn-instance CLIENTE-A
 ip address 10.1.78.7 255.255.255.240
#
```
- ```rd 100:1``` → Define um Route Distinguisher (RD) como 100:1, identificando essa VRF de forma única dentro da rede.

- ```route-target export 100:1``` → Define um Route Target Export como 100:1, permitindo que as rotas dessa VRF sejam compartilhadas com outras VRFs que importem esse valor.

- ```route-target import 100:1``` → Define um Route Target Import como 100:1, permitindo que essa VRF receba rotas de outras VRFs que exportem esse valor.

- ```vrf forwarding CLIENTE-A``` → Define que o IP configurado nessa interface fará parte da VRF CLIENTE-A.

Ok, mas qual a diferença de RD e Route-targets e para que serve isso?

Cada VRF tem um Route Distinguisher (RD), que é configurado para garantir que as rotas dentro de uma VRF sejam únicas, mesmo que os endereços IP sejam os mesmos. O RD é simplesmente um identificador adicionado ao prefixo IP, como por exemplo, ```100:1:172.16.1.0/24```. Isso faz com que o prefixo seja tratado de forma distinta, mesmo que outro roteador tenha a mesma rede ```172.16.1.0/24``` em outra VRF.

Além do RD, os route-targets são usados em redes MPLS VPN (L3VPN) para controlar o compartilhamento de rotas entre VRFs. Eles funcionam como “rótulos” que definem quais rotas podem ser importadas ou exportadas entre diferentes VRFs dentro de uma rede. Se ambas as VRFs usam ```route-target import 100:1```, então elas poderão enxergar as mesmas rotas. Ou seja, os route-targets são usados para definir quem pode falar com quem, garantindo isolamento entre diferentes clientes quando necessário.

 ![](assets/img/posts/post-21/01.png)

Assim, na VRF “global”, terei apenas as redes do Backbone e na VRF CLIENTE-A, por exemplo, apenas a rede do cliente (```172.16.1.0/24``` e ```172.16.2.0/24```) e enlace de PTP para o eBGP.

 ![](assets/img/posts/post-21/02.png)


_VRF "global", apenas rotas do backbone MPLS._

 ![](assets/img/posts/post-21/03.png)


_VRF CLIENTE-A_


Após isso, na configuração do BGP, é necessário o VPNv4.

RT3
```
 address-family vpnv4
  neighbor 10.127.7.7 activate
  neighbor 10.127.7.7 send-community both
```
- ```neighbor 10.127.7.7 send-community both``` → Permite enviar e receber communities BGP (incluindo route-targets, essenciais para MPLS VPN).

E a sessão BGP com o cliente A será feita na própria VRF que foi designada e atribuída a interface com o /30 com o cliente.
```
address-family ipv4 vrf CLIENTE-A
  neighbor 10.1.13.1 remote-as 65020
  neighbor 10.1.13.1 activate
```
Em resumo:

As rotas das VRFs precisam ser diferenciadas dentro da rede MPLS porque podem ter endereços IP iguais. Por isso, utilizamos VPNv4 em vez de BGP IPv4 tradicional. A address-family VPNv4 no BGP permite a troca de rotas contendo o RD (para garantir unicidade) e o RT (para controlar importação e exportação). Esse mecanismo possibilita o transporte isolado das rotas de VRFs, permitindo a criação de VPNs MPLS, onde diferentes clientes ou serviços compartilham a mesma infraestrutura sem interferência de tráfego.

Verificando o cliente A (no cliente foi configurado o allowas-in nas configurações de BGP).

```
show bgp vpnv4 unicast vrf CLIENTE-A
```

 ![](assets/img/posts/post-21/04.png)

```
show bgp vpnv4 unicast vrf CLIENTE-A detail
```

 ![](assets/img/posts/post-21/05.png)


Podemos validar as extended community com os route-targets e até mesmo os labels mpls.

Como o cliente A foi designado para vrf ```100:1``` e permitimos a importação e exportação, elas se falam.

 ![](assets/img/posts/post-21/06.png)


_Cliente A no RJ com traceroute para BA_

Agora com o cliente B será com OSPF. Porém precisarei transportar a rede do cliente que aprendo via OSPF para dentro do iBGP e vice-versa. Com isso vou utilizar o redistribute.

RT3 – CISCO
```
interface GigabitEthernet2
 vrf forwarding CLIENTE-B
 ip address 10.1.23.3 255.255.255.240
 ip ospf 23 area 0

router ospf 23 vrf CLIENTE-B
 redistribute bgp 65010 metric 30
!
router bgp 65010
address-family ipv4 vrf CLIENTE-B
  redistribute ospf 23 route-map ANUNCIO-CLIENTE-B
 exit-address-family

ip prefix-list CLIENTE-B seq 5 permit 172.16.1.0/24 le 32
!
route-map ANUNCIO-CLIENTE-B permit 10
 match ip address prefix-list CLIENTE-B
```
RT7 – HUAWEI
```
interface GigabitEthernet0/0/4
 ip binding vpn-instance CLIENTE-B
 ip address 10.1.79.7 255.255.255.240
 ospf enable 79 area 0.0.0.0
#
ospf 79 vpn-instance CLIENTE-B
 import-route bgp cost 30
 area 0.0.0.0
#
bgp 65010
 ipv4-family vpn-instance CLIENTE-B
  import-route ospf 79 route-policy ANUNCIO-CLIENTE-B
#
ip ip-prefix CLIENTE-B index 10 permit 172.16.2.0 24 greater-equal 24 less-equal 32

route-policy ANUNCIO-CLIENTE-B permit node 10
 if-match ip-prefix CLIENTE-B
```
Dessa forma o que recebe no iBGP é redistribuido no OSPF e o que recebe no OSPF é distribuido no IBGP.

 ![](assets/img/posts/post-21/07.png)

Agora é possível verificar que mesmo esses clientes possuindo as mesmas redes, elas são separadas pela VRF e não se fala entre si.

 ![](assets/img/posts/post-21/08.png)

Podemos ver que no MPLS Forwarding-table as redes ```172.16.1.0/24``` são encaminhadas para interfaces diferentes.

 ![](assets/img/posts/post-21/09.png)


_O ```V``` é referente a VRF_

 ![](assets/img/posts/post-21/10.png)


_Cliente B no RJ com traceroute para BA_


Bônus
Exemplo no DmOS  – DATACOM
```
router bgp 65100
 router-id 10.2.1.21
 address-family ipv4 unicast
 !
 address-family vpnv4 unicast
 !
neighbor 10.2.1.38
 update-source-address 10.2.1.21
 remote-as 65100
 ebgp-multihop 255
 next-hop-self
 address-family ipv4 unicast
 !
 address-family vpnv4 unicast
 !
!
vrf AS6500-AS6501
 address-family ipv4 unicast
 exit-address-family
!
 neighbor 10.100.0.2
  update-source-address 10.100.0.1
  remote-as 6500
  address-family ipv4 unicast
  exit-address-family
   !
  !
 !
!
vrf AS6500-AS6501
 address-family ipv4 unicast
 exit-address-family
!
 neighbor 10.100.0.2
 update-source-address 10.100.0.1
 remote-as 6500
 address-family ipv4 unicast
  exit-address-family
  !
  !
 !
!

vrf AS6500-AS6501
 rd 65100:00
 route-target import 65100:00
!
 route-target export 65100:00
!
dot1q
 vlan 1245
  interface ten-gigabit-ethernet-1/1/13
  !
 !
!

interface l3 TESTE-L3VPN
 vrf AS6500-AS6501
 lower-layer-if vlan 1245
 ipv4 address 10.100.0.1/30
!
```

Conclusão

Com esse lab, consegui demonstrar como VRF, L3VPN, MP-BGP e VPNv4 trabalham juntos para transportar tráfego de diferentes clientes sobre um backbone MPLS. O Cliente A, que tem AS próprio, troca rotas via eBGP, enquanto o Cliente B, que não tem AS, utiliza OSPF dentro da sua VRF, mantendo as redes isoladas mesmo utilizando os mesmos prefixos.

Esse tipo de configuração é amplamente usado por operadoras e grandes redes corporativas para oferecer VPNs de camada 3, garantindo segurança e separação total entre os clientes.