---
title: "07 | Criando um Laboratório de IX / Route Server no PNETLAB"
date: 2024-07-30 00:00:00 -0300
categories: [Tutoriais]
tags: [IX, RouteServer, PNETLAB, BGP]
image:
  path: assets/img/posts/post-07/capa.png
---

Objetivo: Configurar um ambiente de laboratório para testar e aprender sobre Route Servers em um Internet Exchange (IX) usando a plataforma PNETLAB.

Visão Geral:

- Route Servers: São dispositivos que ajudam a simplificar a troca de rotas entre múltiplos fornecedores de serviços de Internet (ISPs) em um Internet Exchange Point (IXP). Eles facilitam o roteamento e reduzem a complexidade ao permitir que ISPs peçam e recebam rotas de outros ISPs através de um único ponto.
- Internet Exchange (IX): Um local onde ISPs e outras redes se conectam e trocam tráfego. Ele ajuda a melhorar a eficiência e reduzir os custos de trânsito ao permitir a troca direta de tráfego entre redes.
- VLAN Bilateral: Uma VLAN bilateral é uma configuração em que duas redes distintas se comunicam através de uma VLAN específica dentro de um IX. Isso é comum em ambientes de IX para permitir a interconexão de diferentes redes sem criar múltiplas VLANs para cada par de redes.

_Observações: Irei utilizar apenas IPv4 e não vou utilizar filtros/communities para simplificar, mas fiquem a vontade para fazerem as boas práticas =D_

Topologia:

![](assets/img/posts/post-07/capa.png)

# Informações do Ambiente IX

## Prefixo IPv4

| Item | Valor |
|-----|------|
| Bloco | 200.0.123.0/22 |
| AS | 65000 |

---

# Dispositivos do Ambiente

## Host 0 — Route Server do IX

| Propriedade | Valor |
|-------------|------|
| Tipo | Server Ubuntu 20.04 LTS |
| Software | FFRouting |
| IP | 200.0.123.254/22 |
| AS | 65000 |

---

## Host 1 — Switch do IX

| Propriedade | Valor |
|-------------|------|
| Tipo | vIOS_l2 |
| Software | vios_l2-ADVENTERPRISEK9-M |

### VLANs

- 100
- 200
- 300
- 400
- 1040

⚠️ Talvez seja necessário **desabilitar o Spanning Tree** para essas VLANs.

---

## Host 2 — Roteador Cisco

| Propriedade | Valor |
|-------------|------|
| Tipo | IOSv |
| Software | VIOS-ADVENTERPRISEK9-M |
| Versão | 15.9(3)M6 |
| IPs | 200.0.123.1/22<br>10.140.0.1/30 |
| Bloco | 100.100.100.0/22 |
| AS | 65100 |

### VLANs

- 100 (IX)
- 1040 (Bilateral com AS65400)

---

## Host 3 — Roteador Mikrotik

| Propriedade | Valor |
|-------------|------|
| Tipo | RouterOS |
| Versão | v6 |
| IP | 200.0.123.2/22 |
| Bloco | 200.200.200.0/21 |
| AS | 65200 |

### VLAN

- 200 (IX)

---

## Host 4 — Roteador Juniper

| Propriedade | Valor |
|-------------|------|
| Tipo | vMX |
| Sistema | JunOS |
| Versão | 21.R1.11 |
| IP | 200.0.123.3/22 |
| Bloco | 133.133.133.0/24 |
| AS | 65400 |

### VLAN

- 300 (IX)

---

## Host 5 — Roteador Huawei

| Propriedade | Valor |
|-------------|------|
| Tipo | NE40E |
| IPs | 200.0.123.4/22<br>10.140.0.2/30 |
| Bloco | 144.144.144.0/22 |
| AS | 65400 |

### VLANs

- 400 (IX)
- 1040 (Bilateral com AS65100)

## Configurações:

Route Server:
Começando pelo Route Server, iremos configurar as vlans dos clientes dentro de uma bridge e essa mesma bridge terá o endereço IP em que todos os participantes do IX fecharão as sessões BGP. O processo será feito com Netplan.
```
root@frrouting:~# nano /etc/netplan/01-netcfg.yaml 
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      mtu: 9000
      dhcp4: no
  vlans:
     vlan.100:
      id: 100
      link: eth0
     vlan.200:
      id: 200
      link: eth0
     vlan.300:
      id: 300
      link: eth0
     vlan.400:
      id: 400
      link: eth0

  bridges:
    br0:
      addresses: [ 200.0.123.254/22 ]
      mtu: 9000
      interfaces:
        - vlan.100
        - vlan.200
        - vlan.300                
        - vlan.400 
root@frrouting:~# netplan apply
```
![](assets/img/posts/post-07/01.png)

Agora iremos configurar o BGP
```
root@frrouting:~# vtysh //entrar no modo CLI Zebra

router bgp 65000
 no bgp ebgp-requires-policy 
 no bgp default ipv4-unicast 
 neighbor 200.0.123.1 remote-as 65100
 neighbor 200.0.123.2 remote-as 65200
 neighbor 200.0.123.3 remote-as 65300
 neighbor 200.0.123.4 remote-as 65400
 !
 address-family ipv4 unicast
  neighbor 200.0.123.1 activate
  neighbor 200.0.123.1 route-server-client 
  neighbor 200.0.123.2 activate
  neighbor 200.0.123.2 route-server-client
  neighbor 200.0.123.3 activate
  neighbor 200.0.123.3 route-server-client
  neighbor 200.0.123.4 activate
  neighbor 200.0.123.4 route-server-client
 exit-address-family
exit
!
end
frrouting# 
```
Algumas observações:

- ```no bgp ebgp-requires-policy```
Necessário desabilitar já que não vou usar filtros, caso não seja desabilitado, não iremos receber ou enviar prefixos conforme imagem abaixo

![](assets/img/posts/post-07/02.png)

- ```no bgp default ipv4-unicast```
Recomendado desabilitar em casos de usar IPv6 também

- ```neighbor 200.0.123.1 route-server-client```
Essa é a configuração que basicamente diz que esse host que estou configurando é um Route Server e os outros peers serão clientes

Switch do IX:

Basicamente iremos configurar as interfaces como trunk e as vlans em suas respectivas portas

``` 
vlan 100,200,300,400,1040
!
interface GigabitEthernet0/0
 switchport trunk allowed vlan 100,1040
 switchport trunk encapsulation dot1q
 switchport mode trunk
 negotiation auto
!
interface GigabitEthernet0/1
 switchport trunk allowed vlan 200
 switchport trunk encapsulation dot1q
 switchport mode trunk
 negotiation auto
!
interface GigabitEthernet0/2
 switchport trunk allowed vlan 300
 switchport trunk encapsulation dot1q
 switchport mode trunk
 negotiation auto
!
interface GigabitEthernet0/3
 switchport trunk allowed vlan 400,1040
 switchport trunk encapsulation dot1q
 switchport mode trunk
 negotiation auto
!
interface GigabitEthernet1/0
 switchport trunk allowed vlan 100,200,300,400
 switchport trunk encapsulation dot1q
 switchport mode trunk
 negotiation auto
!
``` 
![](assets/img/posts/post-07/03.png)

Agora vamos configurar os participantes do IX

Roteador Cisco:
``` 
interface GigabitEthernet0/0.100
 encapsulation dot1Q 100
 ip address 200.0.123.1 255.255.252.0
!
interface GigabitEthernet0/0.1040
 encapsulation dot1Q 1040
 ip address 10.140.0.1 255.255.255.252
!
router bgp 65100
 no bgp enforce-first-as
 neighbor 10.140.0.2 remote-as 65400
 neighbor 200.0.123.254 remote-as 65000
 neighbor 200.0.123.254 description IX-SP
 !
 address-family ipv4
  network 100.100.100.0 mask 255.255.252.0
  neighbor 10.140.0.2 activate
  neighbor 10.140.0.2 soft-reconfiguration inbound
  neighbor 10.140.0.2 route-map IMPORT-AS65400 in
  neighbor 10.140.0.2 route-map EXPORT-TO-AS65400 out
  neighbor 200.0.123.254 activate
  neighbor 200.0.123.254 soft-reconfiguration inbound
 exit-address-family
!         
ip route 100.100.100.0 255.255.252.0 Null0
!
ip prefix-list BLOCO-AS65400 seq 5 permit 144.144.144.0/22 le 24
!
ip prefix-list MEUBLOCO-AS65100 seq 5 permit 100.100.100.0/22 le 24
!
route-map IMPORT-AS65400 permit 10
 match ip address prefix-list BLOCO-AS65400
 set local-preference 150
!
route-map EXPORT-TO-AS65400 permit 10
 match ip address prefix-list MEUBLOCO-AS65100
!
``` 
Algumas observações:

- ``` no bgp enforce-first-as``` 
Normalmente se espera que o peer adicione o seu AS no AS_PATH das rotas que ele anuncia, porém não é o caso e o roteador Cisco não irá aceitar as rotas por isso. Em sessão com Route Server, é necessário desabilitar essa opção.

- ``` ip route 100.100.100.0 255.255.252.0 Null0``` 
Adicionado o prefixo do AS 65100 em blackhole para constar na tabela de roteamento e o BGP conseguir anunciar

- ``` prefix-list```  e ```route-maps``` 
Necessário para o AS 65100 não ser transito dos prefixos do IX para o AS65400.

Aumentado o local-preference para o prefixo ser prioritário na sessão bilateral.

Não é necessário uma regra de deny devido o comportamento padrão do route-map de negar tudo.

Roteador Mikrotik:
``` 
/interface vlan
add interface=ether1 name=ptp-ix-sp vlan-id=200

/ip address
add address=200.0.123.2/22 interface=ptp-ix-sp network=200.0.120.0

/ip route
add distance=1 dst-address=200.200.200.0/21 type=blackhole

/routing bgp instance
set default as=65200 
/routing bgp network
add network=200.200.200.0/21 synchronize=yes
/routing bgp peer
add name=RS-IX-SP remote-address=200.0.123.254 remote-as=65000
``` 
Observação:

- ``` /routing bgp network add network=200.200.200.0/21 synchronize=yes``` 
Caso o syncronize esteja em NO, o RouterOS irá anunciar a rota mesmo que ela não exista na tabela de roteamento (em Blackhole ou em alguma interface)

![](assets/img/posts/post-07/04.png)

Roteador Juniper:
``` 
set interfaces ge-0/0/0 flexible-vlan-tagging
set interfaces ge-0/0/0 unit 300 vlan-id 300
set interfaces ge-0/0/0 unit 300 family inet address 200.0.123.3/22

set protocols bgp group IX-SP local-address 200.0.123.3
set protocols bgp group IX-SP local-as 65300
set protocols bgp group IX-SP neighbor 200.0.123.254 export EXPORT-TO-IX-SP
set protocols bgp group IX-SP neighbor 200.0.123.254 peer-as 65000

set routing-options static route 133.133.133.0/24 discard

set policy-options policy-statement EXPORT-TO-IX-SP term 1 from route-filter 133.133.133.0/22 upto /24
set policy-options policy-statement EXPORT-TO-IX-SP term 1 then accept
set policy-options policy-statement EXPORT-TO-IX-SP term DROP then reject
``` 
Algumas observações:

- ``` set routing-options static route 133.133.133.0/24 discard``` 
Adicionado o prefixo em blackhole

- ``` set policy-options policy-statement EXPORT-TO-IX-SP ``` 

no JunOS não tem a opção network no bgp, para exportar a rede tem que ser por meio de uma policy. Por padrão ela aceita tudo, por isso temos criar o reject no final

Roteador Huawei:
```  
interface Ethernet1/0/0.400
 vlan-type dot1q 400
 ip address 200.0.123.4 255.255.252.0
#
interface Ethernet1/0/0.1040
 vlan-type dot1q 1040
 ip address 10.140.0.2 255.255.255.252
#
bgp 65400
 router-id 10.4.4.4
 undo check-first-as
 peer 10.140.0.1 as-number 65100
 peer 200.0.123.254 as-number 65000
 #
 ipv4-family unicast
  network 144.144.144.0 255.255.252.0
  peer 10.140.0.1 enable
  peer 10.140.0.1 route-policy IMPORT-AS65100 import
  peer 10.140.0.1 route-policy EXPORT-TO-AS65100 export
  peer 200.0.123.254 enable
#
route-policy EXPORT-TO-AS65100 permit node 10
 if-match ip-prefix MEUBLOCO-AS65400
#
route-policy IMPORT-AS65100 permit node 10
 if-match ip-prefix BLOCO-AS65100
 apply local-preference 150
#
ip ip-prefix BLOCO-AS65100 index 10 permit 100.100.100.0 22
ip ip-prefix MEUBLOCO-AS65400 index 10 permit 144.144.144.0 22
#
ip route-static 144.144.144.0 255.255.252.0 NULL0
#
```
Algumas observações:

- ```undo check-first-as```
Normalmente se espera que o peer adicione o seu AS no AS_PATH das rotas que ele anuncia, porém não é o caso e o roteador Cisco não irá aceitar as rotas por isso. Em sessão com Route Server, é necessário desabilitar essa opção.

- ```route-policy``` e ```ip-prefix```
Necessário para o AS 65400 não ser transito dos prefixos do IX para o AS65100.
Aumentado o local-preference para o prefixo ser prioritário na sessão bilateral.
Não é necessário uma regra de deny devido o comportamento padrão do route-map de negar tudo.

Verificando as tabelas:
Route Server:
![](assets/img/posts/post-07/05.png)

Roteador Cisco:
![](assets/img/posts/post-07/06.png)

Roteador Mikrotik:
![](assets/img/posts/post-07/07.png)

Roteador Juniper:
![](assets/img/posts/post-07/08.png)

Roteador Huawei:
![](assets/img/posts/post-07/09.png)

Bônus:
Configurando com Bird (Internet Routing Daemon)
O BIRD é mais comum em ser utilizado na maioria dos IXs (em breve irei fazer um post sobre)

Exemplo de configuração no /etc/bird/bird.conf (após configurar, reinicie o serviço – ```systemctl restart bird```)
```
protocol bgp peer200 { 
description "AS 200"; 
local as 1000; 
neighbor 200.0.123.2 as 200; 
rs client;  
import all; 
export all;
keepalive time 60;
hold time 600;
}
```
- ```rs client```
Se torna um route-server e trata o neighbor como server client

- ```hold time```
Aconselho aumentar acima do padrão pois já tive problemas com hold time expired

Verificando no CLI birdc

![](assets/img/posts/post-07/10.png)

Conclusão:
O ambiente IX configurado no PNETLAB demonstra um teste de interconexão entre diferentes redes A configuração inclui um Route Server centralizado para facilitar a troca de rotas entre os roteadores, um Switch para gerenciar múltiplas VLANs e diferentes roteadores representando diversos fornecedores com ASNs distintos.

Os dispositivos foram configurados com IPs e VLANs específicos para garantir a comunicação eficiente e segura entre as redes. A utilização de VLANs bilaterais, como mostrado com a VLAN 1040, permite a interconexão direta entre ASNs diferentes, simulando cenários reais de um Internet Exchange.

Essa configuração não só ajuda a entender como funcionam as trocas de rotas em um ambiente IX, mas também proporciona uma base sólida para testes e aprendizado sobre o gerenciamento de redes complexas e políticas de roteamento.

Autor: Jefferson Raimon

Referências:
[Bird](https://bird.network.cz/)
[Intro to BGP](https://natecatelli.com/posts/intro-to-bgp/)