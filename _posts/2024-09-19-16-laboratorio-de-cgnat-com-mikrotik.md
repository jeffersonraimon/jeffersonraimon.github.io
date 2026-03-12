---
title: "16 | Laboratório de CGNAT com Mikrotik"
date: 2024-09-19 00:00:00 -0300
categories: [Tutoriais]
tags: [CGNAT, Mikrotik, NAT, Lab]
image:
  path: assets/img/posts/post-16/capa.png
---

Objetivo: Configurar um ambiente de laboratório simples para testar e aprender sobre CGNAT e PBR com Mikrotik.

Topologia

![](assets/img/posts/post-16/capa.png)

## Informações dos Equipamentos

---

### PC Cliente

| Propriedade | Valor |
|-------------|------|
| Tipo | VM Linux Debian |
| Função | Discar PPPoE |
| Endereçamento | Recebe IP do bloco CGNAT |

---

## Autenticador

### MK-RT

| Propriedade | Valor |
|-------------|------|
| Sistema | RouterOS CHR |
| Versão | 6.49.10 LTS |
| Loopback PPPoE Server | 10.253.1.1 |

### VLANs

| VLAN | Função | Rede |
|-----|------|------|
| 100 | Clientes PPPoE | 100.64.128.0/24 |
| 210 | PTP com CGNAT / iBGP | 10.200.10.2/30 |
| 3100 | PTP com Roteador de Borda / iBGP | 10.100.10.2/30 |

---

## CGNAT

### MK-RT-CGNAT

| Propriedade | Valor |
|-------------|------|
| Sistema | RouterOS CHR |
| Versão | 6.49.10 LTS |
| Prefixo (Simulando público) | 100.111.111.0/24 |

### VLANs

| VLAN | Função | Rede |
|-----|------|------|
| 210 | PTP com Autenticador / iBGP | 10.200.10.1/30 |
| 3000 | PTP com Roteador de Borda / iBGP | 172.31.192.38/30 |

---

## Roteador de Borda

### JUNIPER-VMX

| Propriedade | Valor |
|-------------|------|
| Sistema | Juniper vMX |
| vCP | vmxvcp-21.1R1.11 |
| vFP | vmxvfp-21.1R1.11 |
| AS | 155 |

### VLANs

| VLAN | Função | Rede |
|-----|------|------|
| 3100 | PTP com Autenticador / iBGP | 10.100.10.1/30 |
| 3000 | PTP com CGNAT / iBGP | 172.31.192.37/30 |

### eBGP

| Parâmetro | Valor |
|----------|------|
| Peering | AS200 |
| Link PTP | 10.155.200.1/30 |

---

## Switch

### Cisco IOL

| Propriedade | Valor |
|-------------|------|
| Função | LAN do AS155 |

### Configuração

- Portas em **trunk**
- VLANs permitidas:
  - 100
  - 210
  - 3000
  - 3100

- Porta conectada ao **cliente PPPoE em modo access**

---

## Roteador AS200

### Cisco IOSv

| Propriedade | Valor |
|-------------|------|
| Imagem | vios-adventerprisek9-m.SPA.159-3.M6 |
| AS | 200 |

### eBGP

| Parâmetro | Valor |
|----------|------|
| Peering | AS155 |
| Link PTP | 10.155.200.2/30 |

### Prefixo anunciado

- 192.2.10.0/24


---

## Server AS200

| Propriedade | Valor |
|-------------|------|
| Serviço | Apache |
| IP | 192.2.10.2 |

Autenticador:
Começando pelo autenticador, irei configurar a loopback, vlans, servidor pppoe, ibgp, filtros etc.
```
/system identity set name=AUTENT

/interface bridge add name=loopback protocol-mode=none

/interface vlan add interface=ether1 name=pppoe vlan-id=100
/interface vlan add interface=ether1 name=ptp-cgnat vlan-id=210
/interface vlan add interface=ether1 name=ptp-rt vlan-id=3100

/ip address add address=10.100.10.2/30 interface=ptp-rt network=10.100.10.0
/ip address add address=10.200.10.2/30 interface=ptp-cgnat network=10.200.10.0
/ip address add address=10.253.1.1 interface=loopback network=10.253.1.1

/ip route add distance=1 dst-address=100.64.128.0/24 type=blackhole

/ip pool add name=cliente-pppoe ranges=100.64.128.1-100.64.128.254
/ppp profile set *0 dns-server=10.253.1.1 local-address=10.253.1.1 remote-address=cliente-pppoe
/ppp secret add name=cliente password=cliente
/interface pppoe-server server add disabled=no interface=pppoe one-session-per-host=yes service-name=pppoe-server

/routing bgp instance set default as=155 router-id=10.253.1.1
/routing bgp network add network=100.64.128.0/24
/routing bgp peer add in-filter=IN name=ibgp-cgnat out-filter=OUT remote-address=10.200.10.1 remote-as=155
/routing bgp peer add name=ibgp-rt in-filter=IN-DEFAULT out-filter=OUT-DISCARD remote-address=10.100.10.1 remote-as=155

/routing filter add action=accept chain=IN-DEFAULT prefix=0.0.0.0/0
/routing filter add action=discard chain=IN-DEFAULT
/routing filter add action=discard chain=IN
/routing filter add action=accept chain=OUT prefix=100.64.128.0/24
/routing filter add action=discard chain=OUT
/routing filter add action=discard chain=OUT-DISCARD
/ip firewall connection tracking set enabled=no
```
Como podemos ver nos filtros, o autenticador apenas recebe rota default do roteador de borda e anuncia a rede ```100.64.128.0/24``` para o mesmo e também para a caixa do CGNAT (e não recebe nada via BGP). Também é desabilitado a conntrack, pois quem irá realizar tal mapeamento das conexões é o CGNAT. Foi colocado o prefixo do CGNAT em blackhole para evitar loop de roteamento. Veja mais em: <https://www.linkedin.com/pulse/como-identificar-e-corrigir-loops-de-roteamento-evandro-alves-pereira/>.

Agora vamos para o que realmente importa:

```
/ip route add dst-address=0.0.0.0/0 distance=1 gateway=10.200.10.1 routing-mark=CGNAT
/ip route rule add action=lookup-only-in-table src-address=100.64.128.0/24 table=CGNAT
```
Aqui entra a PBR (Policy Based Routing). Esses comandos configuram o roteador para direcionar todo o tráfego (qualquer destino) para a caixa do CGNAT se a origem do tráfego estiver na faixa ```100.64.128.0/24```, usando uma tabela de roteamento específica chamada “CGNAT”.

- ```routing-mark=CGNAT```: Marca a rota com “CGNAT”, o que pode ser usado para distinguir essa rota em políticas de roteamento.

- ```action=lookup-only-in-table```: Essa ação instrui o roteador APENAS buscar a tabela de roteamento especificada, sem buscar na main caso não encontre nada nessa tabela.

- ```table=CGNAT```: Especifica que a busca deve ser feita na tabela de roteamento marcada como “CGNAT”.

![](assets/img/posts/post-16/01.png)

CGNAT:

Implementação simples do CGNAT utilizando o script do [Remontti](https://cgnat.remontti.com.br/). Anunciamos o prefixo ```100.111.111.0/24``` (também em blackhole) para o roteador de borda e recebemos o prefixo ```100.64.128.0./24``` do autenticador com o iBGP.
```
/system identity set name=CGNAT

/interface bridge add name=loopback protocol-mode=none

/interface vlan add interface=ether1 name=ptp-autent vlan-id=210
/interface vlan add interface=ether1 name=ptp-rt vlan-id=3000

/ip address add address=172.31.192.38/30 interface=ptp-rt network=172.31.192.36
/ip address add address=10.200.10.1/30 interface=ptp-autent network=10.200.10.0
/ip address add address=10.252.1.1 interface=loopback network=10.252.1.1

/routing bgp instance set default as=155 router-id=10.252.1.1
/routing bgp network add network=100.111.111.1/32 synchronize=no
/routing bgp network add network=100.111.111.0/24 synchronize=no
/routing bgp peer add in-filter=IN-DEFAULT name=ibgp-rt out-filter=OUT-PREFIX remote-address=172.31.192.37 remote-as=155
/routing bgp peer add in-filter=IN-AUTENT name=ibgp-autent out-filter=OUT remote-address=10.200.10.2 remote-as=155


/ip firewall filter add action=fasttrack-connection chain=forward connection-state=established,related
/ip firewall filter add action=accept chain=forward connection-state=established,related

/ip firewall nat add action=src-nat chain=srcnat disabled=yes src-address=100.64.128.0/24 to-addresses=100.111.111.1
/ip firewall nat add action=jump chain=srcnat jump-target=CGNAT_0 src-address=100.64.128.0/24
/ip firewall nat add action=netmap chain=CGNAT_0 protocol=tcp src-address=100.64.128.0/24 to-addresses=100.111.111.0/24 to-ports=1024-3039
/ip firewall nat add action=netmap chain=CGNAT_0 protocol=udp src-address=100.64.128.0/24 to-addresses=100.111.111.0/24 to-ports=1024-3039
/ip firewall nat add action=netmap chain=CGNAT_0 src-address=100.64.128.0/24 to-addresses=100.111.111.0/24

/ip route add distance=1 dst-address=100.111.111.0/24 type=blackhole

/routing filter add action=accept chain=IN prefix=100.64.128.0/24
/routing filter add action=discard chain=IN
/routing filter add action=accept chain=OUT-PREFIX prefix=100.111.111.0/24
/routing filter add action=discard chain=OUT-PREFIX
/routing filter add action=accept chain=IN-DEFAULT prefix=0.0.0.0/0
/routing filter add action=discard chain=IN-DEFAULT
/routing filter add action=accept chain=IN-AUTENT prefix=100.64.128.0/24
/routing filter add action=discard chain=IN-AUTENT
/routing filter add action=discard chain=OUT
```
![](assets/img/posts/post-16/02.png)


Roteador de Borda:

No roteador de borda, com iBGP basicamente recebemos o prefixo ```100.111.111.0/24``` do CGNAT, e entregamos rota default para o autenticador e o CGNAT e recebemos via eBGP o prefixo do AS200 (para fazermos o teste) e anunciamos o nosso prefixo “publico”.

```
delete chassis auto-image-upgrade

set interfaces ge-0/0/0 flexible-vlan-tagging
set interfaces ge-0/0/0 unit 3000 description ptp-cgnat
set interfaces ge-0/0/0 unit 3000 vlan-id 3000
set interfaces ge-0/0/0 unit 3000 family inet address 172.31.192.37/30
set interfaces ge-0/0/0 unit 3100 description ptp-autent
set interfaces ge-0/0/0 unit 3100 vlan-id 3100
set interfaces ge-0/0/0 unit 3100 family inet address 10.100.10.1/30
set interfaces ge-0/0/1 unit 0 description ptp-as200
set interfaces ge-0/0/1 unit 0 family inet address 10.155.200.1/30
set interfaces ge-0/0/2 unit 0 family inet address 172.20.0.1/30

set policy-options policy-statement ANUNCIO-AS200 term 1 from protocol bgp
set policy-options policy-statement ANUNCIO-AS200 term 1 from route-filter 192.2.10.0/24 exact
set policy-options policy-statement ANUNCIO-AS200 term 1 then accept
set policy-options policy-statement ANUNCIO-AS200 term rjc then reject
set policy-options policy-statement ANUNCIO-EXT term 1 from route-filter 100.111.111.0/24 exact
set policy-options policy-statement ANUNCIO-EXT term 1 then accept
set policy-options policy-statement ANUNCIO-EXT term rjc then reject
set policy-options policy-statement EXPORT-DEFAULT term 1 from route-filter 0.0.0.0/0 exact
set policy-options policy-statement EXPORT-DEFAULT term 1 then accept
set policy-options policy-statement EXPORT-DEFAULT term rjc then reject

set protocols bgp group EXT type external
set protocols bgp group EXT local-address 10.155.200.1
set protocols bgp group EXT local-as 155
set protocols bgp group EXT neighbor 10.155.200.2 import ANUNCIO-AS200
set protocols bgp group EXT neighbor 10.155.200.2 export ANUNCIO-EXT
set protocols bgp group EXT neighbor 10.155.200.2 peer-as 200
set protocols bgp group INT type internal
set protocols bgp group INT neighbor 10.100.10.2 description AUTENT
set protocols bgp group INT neighbor 10.100.10.2 import IMPORT-autent
set protocols bgp group INT neighbor 10.100.10.2 export EXPORT-DEFAULT
set protocols bgp group INT neighbor 10.100.10.2 peer-as 155
set protocols bgp group INT neighbor 172.31.192.38 description CGNAT
set protocols bgp group INT neighbor 172.31.192.38 export EXPORT-DEFAULT
set protocols bgp group INT neighbor 172.31.192.38 peer-as 155

set routing-options autonomous-system 155
set routing-options generate route 0.0.0.0/0 discard
```
![](assets/img/posts/post-16/03.png)
![](assets/img/posts/post-16/04.png)


Roteador AS200:

Fiz uma configuração bem simples apenas para testar a conectividade. Não fiz filtros com o route-map.
```
interface GigabitEthernet0/0
 ip address 192.2.10.1 255.255.255.0
!
interface GigabitEthernet0/1
 ip address 10.155.200.2 255.255.255.252
!
router bgp 200
 no bgp default ipv4-unicast
 neighbor 10.155.200.1 remote-as 155
 !
 address-family ipv4
  network 192.2.10.0
  neighbor 10.155.200.1 activate
 exit-address-family
 !
!
```

![](assets/img/posts/post-16/05.png)

## Testes

PC Cliente:

Fiz a discagem PPPoE e obtenho o IP ```100.64.128.254```.

![](assets/img/posts/post-16/06.png)

Utilizando o MTR podemos ver que temos conectividade com o servidor do AS200. É possivel notar o efeito da PBR, que desviou o trafego para o CGNAT (```IP 10.200.10.1```).

![](assets/img/posts/post-16/07.png)

E por ultimo conseguimos acesso a pagina WEB do servidor.

![](assets/img/posts/post-16/08.png)

É possivel observar trafego nas regras criadas no CGNAT.

![](assets/img/posts/post-16/09.png)

Servidor:

Podemos ver no log que o acesso foi realizado com o IP ```100.111.111.254```.

![](assets/img/posts/post-16/10.png)

Como esperado não é possível o servidor realizar ping para o cliente, pois a rede ```100.64.128.0/24``` não é anunciada para o AS200 e o prefixo ```100.111.111.0/24``` esta em blackhole na caixa do CGNAT.

![](assets/img/posts/post-16/11.png)
![](assets/img/posts/post-16/12.png)

Fim!