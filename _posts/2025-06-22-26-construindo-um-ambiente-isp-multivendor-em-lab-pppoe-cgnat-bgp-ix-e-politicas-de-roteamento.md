---
title: "26 | Construindo um Ambiente ISP Multivendor em Lab: PPPOE + CGNAT, BGP, IX e Políticas de Roteamento"
date: 2025-06-22 00:00:00 -0300
categories: [Tutoriais]
tags: [ISP, PPPOE, CGNAT, BGP, IX, Routing]
image:
  path: assets/img/posts/post-26/capa.png
---

O laboratório da vez foi simulando um ambiente ISP real. Neste, realizo configurações desde vlans até politicas de roteamento.

Não explicarei em detalhes o processo, mas gravei esse video de 2h configurando na integra e postei no Youtube.


{% include embed/youtube.html id='ddYO7_1bwJA' %}


Segue abaixo informações das imagens utilizadas e todas as configurações.

Imagens

| Nome | Versao |
| -------| ------ |
| PC-CLIENTE | linux-debian-10 |
| WEB SERVER, BIRD, DNS SERVER | linux-ubuntu-server-20.04-lts |
| ROTEADORES CISCO | viosl2-adventerprisek9-m.ssa.high_iron_20200929 |
| ROTEADORES MIKROTIK | chr-6.49.10-lts e chr-7.18.2 |
| ROTEADOR HUAWEI | huaweine40e-ne40e |
| ROTEADORES JUNIPER | vmx-14.1R4.8-domestic |
| ROTEADOR VYOS | vyos-1.4.0-rolling-amd64-202204040643 |
| SWITCH HUAWEI | huaweice12800-ce12800 |
| SWITCHES CISCO | vios-adventerprisek9-m.SPA.159-3.M6 |

Todas as imagens foram adquiridas por meio do ishare2 (https://ishare2.sh/)

Configurações
MIKROTIK-CGNAT
```
/interface bridge add name=loopback
/interface vlan add interface=ether1 name=ptp-autent vlan-id=3013
/interface vlan add interface=ether1 name=ptp-r1 vlan-id=1014
/interface vlan add interface=ether1 name=ptp-r2 vlan-id=2014
/routing bgp instance set default as=10
/ip address add address=10.10.14.2/30 interface=ptp-r1 network=10.10.14.0
/ip address add address=10.10.24.2/30 interface=ptp-r2 network=10.10.24.0
/ip address add address=10.10.34.2/30 interface=ptp-autent network=10.10.34.0
/ip address add address=10.10.4.4 interface=loopback network=10.10.4.4
/ip firewall filter add action=fasttrack-connection chain=forward connection-state=established,related
/ip firewall filter add action=accept chain=forward connection-state=established,related
/ip firewall nat add action=jump chain=srcnat jump-target=CGNAT_0 src-address=100.64.0.0/24
/ip firewall nat add action=netmap chain=CGNAT_0 protocol=tcp src-address=100.64.0.0/24 to-addresses=111.1.2.0/24 to-ports=1024-3039
/ip firewall nat add action=netmap chain=CGNAT_0 protocol=udp src-address=100.64.0.0/24 to-addresses=111.1.2.0/24 to-ports=1024-3039
/ip firewall nat add action=netmap chain=CGNAT_0 src-address=100.64.0.0/24 to-addresses=111.1.2.0/24
/ip route add distance=1 dst-address=111.1.2.0/24 type=blackhole
/routing bgp network add network=111.1.2.0/24
/routing bgp peer add in-filter=IN-DEFAULT name=ibgp-rt1 out-filter=OUT-PREFIX remote-address=10.10.14.1 remote-as=10
/routing bgp peer add in-filter=IN2-DEFAULT name=ibgp-rt2 out-filter=OUT-PREFIX remote-address=10.10.24.1 remote-as=10
/routing bgp peer add in-filter=IN name=ibgp-autent out-filter=OUT-DISCARD remote-address=10.10.34.1 remote-as=10
/routing filter add action=accept chain=IN prefix=100.64.0.0/24
/routing filter add action=discard chain=IN
/routing filter add action=accept chain=OUT-PREFIX prefix=111.1.2.0/24
/routing filter add action=discard chain=OUT-PREFIX
/routing filter add action=accept chain=IN-DEFAULT prefix=0.0.0.0/0 set-bgp-local-pref=110
/routing filter add action=discard chain=IN-DEFAULT
/routing filter add action=accept chain=IN2-DEFAULT prefix=0.0.0.0/0
/routing filter add action=discard chain=IN2-DEFAULT
/routing filter add action=discard chain=OUT-DISCARD
/system identity set name=CGNAT
```
CISCO-SW-1
```
hostname SW-1
!
vlan 1012
 name PTP-RTS
!
vlan 1013
 name PTP-RT1-AUTENT
!
vlan 1014
 name PTP-RT1-CGNAT
!
vlan 1111
 name PTP-RT1-CLIENTE
!
vlan 1300
 name PTP-RT1-SERVERS
!
vlan 2013
 name PTP-RT2-AUTENT
!
vlan 2014
 name PTP-RT2-CGNAT
!
vlan 2222
 name PTP-RT2-CLIENTE
!
!
interface Port-channel1
 no shutdown
 switchport trunk allowed vlan 1013,1014,2013,2014
 switchport trunk encapsulation dot1q
 switchport mode trunk
!
interface GigabitEthernet0/0
 no shutdown
 switchport trunk allowed vlan 1013,1014,2013,2014
 switchport trunk encapsulation dot1q
 switchport mode trunk
 negotiation auto
 channel-group 1 mode active
!
interface GigabitEthernet0/1
 no shutdown
 switchport trunk allowed vlan 1013,1014,2013,2014
 switchport trunk encapsulation dot1q
 switchport mode trunk
 negotiation auto
 channel-group 1 mode active
!
interface GigabitEthernet0/2
 no shutdown
 description PTP-RT1
 switchport trunk allowed vlan 1012-1014,1111,1300
 switchport trunk encapsulation dot1q
 switchport mode trunk
 negotiation auto
!
interface GigabitEthernet0/3
 no shutdown
 description PTP-RT2
 switchport trunk allowed vlan 1012,2013,2014,2222
 switchport trunk encapsulation dot1q
 switchport mode trunk
 negotiation auto
!
interface GigabitEthernet1/0
 no shutdown
 description SERVER-DNS
 switchport access vlan 1300
 switchport mode access
 negotiation auto
!
interface GigabitEthernet1/1
 no shutdown
 description PTP-CLIENTE
 switchport trunk allowed vlan 1111,2222
 switchport trunk encapsulation dot1q
 switchport mode trunk
 negotiation auto
!
```
HUAWEI-SW-2
```
sysname SW-2
#
vlan batch 1013 to 1014 2013 to 2014 3013 
#
interface Eth-Trunk1
 mode lacp-static
 port link-type trunk
 port trunk allow-pass vlan 1013 to 1014 2013 to 2014
#
interface GE1/0/0
 undo shutdown
 eth-trunk 1
#
interface GE1/0/1
 undo shutdown
 eth-trunk 1
#
interface GE1/0/2
 description AUTENT
 undo shutdown
 port link-type trunk
 port trunk allow-pass vlan 1013 2013 3013 
#
interface GE1/0/3
 description CGNAT
 undo shutdown
 port link-type trunk
 port trunk allow-pass vlan 1014 2014 3013
#
interface GE1/0/4
 description CLIENTE-PPPOE
 undo shutdown
 port default vlan 4000
#
```
JUNIPER-RT1-AS10
```
set system host-name RT1-AS10
set interfaces ge-0/0/0 vlan-tagging
set interfaces ge-0/0/0 unit 1012 vlan-id 1012
set interfaces ge-0/0/0 unit 1012 family inet address 10.10.12.1/30
set interfaces ge-0/0/0 unit 1013 vlan-id 1013
set interfaces ge-0/0/0 unit 1013 family inet address 10.10.13.1/30
set interfaces ge-0/0/0 unit 1014 vlan-id 1014
set interfaces ge-0/0/0 unit 1014 family inet address 10.10.14.1/30
set interfaces ge-0/0/0 unit 1111 vlan-id 1111
set interfaces ge-0/0/0 unit 1111 family inet address 111.1.0.101/30
set interfaces ge-0/0/0 unit 1300 vlan-id 1300
set interfaces ge-0/0/0 unit 1300 family inet address 111.1.1.1/24
set interfaces ge-0/0/1 unit 0 description PTP-AS20
set interfaces ge-0/0/1 unit 0 family inet address 10.10.20.1/30
set routing-options static route 111.1.0.0/22 discard
set routing-options generate route 0.0.0.0/0 discard
set routing-options autonomous-system 10
set protocols bgp group UPSTREAM neighbor 10.10.20.2 import IMPORT-ALL
set protocols bgp group UPSTREAM neighbor 10.10.20.2 export EXPORT-UPSTREAM
set protocols bgp group UPSTREAM neighbor 10.10.20.2 peer-as 20
set protocols bgp group CLIENTES neighbor 111.1.0.102 import IMPORT-CLIENTE-60
set protocols bgp group CLIENTES neighbor 111.1.0.102 export EXPORT-DEFAULT-ROUTE+FULL
set protocols bgp group CLIENTES neighbor 111.1.0.102 peer-as 60
set protocols bgp group IBGP neighbor 10.10.12.2 description PTP-RT2-AS10
set protocols bgp group IBGP neighbor 10.10.12.2 import IMPORT-ALL
set protocols bgp group IBGP neighbor 10.10.12.2 export EXPORT-IBGP-ALL
set protocols bgp group IBGP neighbor 10.10.12.2 peer-as 10
set protocols bgp group IBGP neighbor 10.10.13.2 description AUTENT
set protocols bgp group IBGP neighbor 10.10.13.2 import DROP-ALL
set protocols bgp group IBGP neighbor 10.10.13.2 export EXPORT-DEFAULT-ROUTE
set protocols bgp group IBGP neighbor 10.10.13.2 peer-as 10
set protocols bgp group IBGP neighbor 10.10.14.2 description CGNAT
set protocols bgp group IBGP neighbor 10.10.14.2 import IMPORT-PRX-CGNAT
set protocols bgp group IBGP neighbor 10.10.14.2 export EXPORT-DEFAULT-ROUTE
set protocols bgp group IBGP neighbor 10.10.14.2 peer-as 10
set policy-options policy-statement DROP-ALL term reject then reject
set policy-options policy-statement EXPORT-DEFAULT-ROUTE term 1 from route-filter 0.0.0.0/0 exact
set policy-options policy-statement EXPORT-DEFAULT-ROUTE term 1 then accept
set policy-options policy-statement EXPORT-DEFAULT-ROUTE term reject then reject
set policy-options policy-statement EXPORT-DEFAULT-ROUTE+FULL term 1 from route-filter 0.0.0.0/0 upto /24
set policy-options policy-statement EXPORT-DEFAULT-ROUTE+FULL term 1 then accept
set policy-options policy-statement EXPORT-DEFAULT-ROUTE+FULL term reject then reject
set policy-options policy-statement EXPORT-IBGP-ALL term 1 from protocol bgp
set policy-options policy-statement EXPORT-IBGP-ALL term 1 then next-hop self
set policy-options policy-statement EXPORT-IBGP-ALL term 1 then accept
set policy-options policy-statement EXPORT-IBGP-ALL term 2 from route-filter 111.1.0.0/22 upto /24
set policy-options policy-statement EXPORT-IBGP-ALL term 2 then accept
set policy-options policy-statement EXPORT-IBGP-ALL term reject then reject
set policy-options policy-statement EXPORT-UPSTREAM term 1 from route-filter 111.1.0.0/22 upto /24
set policy-options policy-statement EXPORT-UPSTREAM term 1 then accept
set policy-options policy-statement EXPORT-UPSTREAM term 2 from community CLIENTE-60
set policy-options policy-statement EXPORT-UPSTREAM term 2 then accept
set policy-options policy-statement EXPORT-UPSTREAM term reject then reject
set policy-options policy-statement IMPORT-ALL term 1 from protocol bgp
set policy-options policy-statement IMPORT-ALL term 1 then accept
set policy-options policy-statement IMPORT-ALL term reject then reject
set policy-options policy-statement IMPORT-CLIENTE-60 term 1 from as-path CLIENTE-AS60
set policy-options policy-statement IMPORT-CLIENTE-60 term 1 then community set CLIENTE-60
set policy-options policy-statement IMPORT-CLIENTE-60 term 1 then accept
set policy-options policy-statement IMPORT-CLIENTE-60 term reject then reject
set policy-options policy-statement IMPORT-PRX-CGNAT term 1 from route-filter 111.1.2.0/24 exact
set policy-options policy-statement IMPORT-PRX-CGNAT term 1 then accept
set policy-options policy-statement IMPORT-PRX-CGNAT term reject then reject
set policy-options community CLIENTE-60 members 10:60
set policy-options as-path CLIENTE-AS60 ".*60$"
```
JUNIPER-RT2-AS10
```
set system host-name RT2-AS10
set interfaces ge-0/0/0 vlan-tagging
set interfaces ge-0/0/0 unit 1012 vlan-id 1012
set interfaces ge-0/0/0 unit 1012 family inet address 10.10.12.2/30
set interfaces ge-0/0/0 unit 2013 vlan-id 2013
set interfaces ge-0/0/0 unit 2013 family inet address 10.10.23.1/30
set interfaces ge-0/0/0 unit 2014 vlan-id 2014
set interfaces ge-0/0/0 unit 2014 family inet address 10.10.24.1/30
set interfaces ge-0/0/0 unit 2222 vlan-id 2222
set interfaces ge-0/0/0 unit 2222 family inet address 111.1.0.201/30
set interfaces ge-0/0/1 vlan-tagging
set interfaces ge-0/0/1 unit 1070 description BILATERAL-AS70
set interfaces ge-0/0/1 unit 1070 vlan-id 1070
set interfaces ge-0/0/1 unit 1070 family inet address 10.10.70.1/30
set interfaces ge-0/0/1 unit 3000 description IX-RS
set interfaces ge-0/0/1 unit 3000 vlan-id 3000
set interfaces ge-0/0/1 unit 3000 family inet address 123.123.0.10/24
set interfaces ge-0/0/2 unit 0 description PTP-AS40
set interfaces ge-0/0/2 unit 0 family inet address 10.10.40.1/30
set routing-options generate route 0.0.0.0/0 discard
set routing-options autonomous-system 10
set protocols bgp group CLIENTES neighbor 111.1.0.202 import IMPORT-CLIENTE-60
set protocols bgp group CLIENTES neighbor 111.1.0.202 export EXPORT-DEFAULT-ROUTE+FULL
set protocols bgp group CLIENTES neighbor 111.1.0.202 peer-as 60
set protocols bgp group UPSTREAM neighbor 123.123.0.254 import IMPORT-IX-ALL
set protocols bgp group UPSTREAM neighbor 123.123.0.254 export EXPORT-AS10-IX
set protocols bgp group UPSTREAM neighbor 123.123.0.254 peer-as 123
set protocols bgp group UPSTREAM neighbor 10.10.70.2 import IMPORT-AS70-IX
set protocols bgp group UPSTREAM neighbor 10.10.70.2 export EXPORT-AS10-IX
set protocols bgp group UPSTREAM neighbor 10.10.70.2 peer-as 70
set protocols bgp group UPSTREAM neighbor 10.10.40.2 import IMPORT-ALL
set protocols bgp group UPSTREAM neighbor 10.10.40.2 export EXPORT-UPSTREAM
set protocols bgp group UPSTREAM neighbor 10.10.40.2 peer-as 40
set protocols bgp group IBGP neighbor 10.10.12.1 import IMPORT-ALL
set protocols bgp group IBGP neighbor 10.10.12.1 export EXPORT-IBGP-ALL
set protocols bgp group IBGP neighbor 10.10.12.1 peer-as 10
set protocols bgp group IBGP neighbor 10.10.23.2 import DROP-ALL
set protocols bgp group IBGP neighbor 10.10.23.2 export EXPORT-DEFAULT-ROUTE
set protocols bgp group IBGP neighbor 10.10.23.2 peer-as 10
set protocols bgp group IBGP neighbor 10.10.24.2 import IMPORT-PRX-CGNAT
set protocols bgp group IBGP neighbor 10.10.24.2 export EXPORT-DEFAULT-ROUTE
set protocols bgp group IBGP neighbor 10.10.24.2 peer-as 10
set policy-options policy-statement DROP-ALL term reject then reject
set policy-options policy-statement EXPORT-AS10-IX term 1 from route-filter 111.1.0.0/22 upto /24
set policy-options policy-statement EXPORT-AS10-IX term 1 then accept
set policy-options policy-statement EXPORT-AS10-IX term reject then reject
set policy-options policy-statement EXPORT-DEFAULT-ROUTE term 1 from route-filter 0.0.0.0/0 exact
set policy-options policy-statement EXPORT-DEFAULT-ROUTE term 1 then accept
set policy-options policy-statement EXPORT-DEFAULT-ROUTE term reject then reject
set policy-options policy-statement EXPORT-DEFAULT-ROUTE+FULL term 1 from route-filter 0.0.0.0/0 upto /24
set policy-options policy-statement EXPORT-DEFAULT-ROUTE+FULL term 1 then accept
set policy-options policy-statement EXPORT-DEFAULT-ROUTE+FULL term reject then reject
set policy-options policy-statement EXPORT-IBGP-ALL term 1 from protocol bgp
set policy-options policy-statement EXPORT-IBGP-ALL term 1 then next-hop self
set policy-options policy-statement EXPORT-IBGP-ALL term 1 then accept
set policy-options policy-statement EXPORT-IBGP-ALL term 2 from route-filter 111.1.0.0/22 upto /24
set policy-options policy-statement EXPORT-IBGP-ALL term 2 then accept
set policy-options policy-statement EXPORT-IBGP-ALL term reject then reject
set policy-options policy-statement EXPORT-UPSTREAM term 1 from route-filter 111.1.0.0/22 upto /24
set policy-options policy-statement EXPORT-UPSTREAM term 1 then accept
set policy-options policy-statement EXPORT-UPSTREAM term 2 from community CLIENTE-60
set policy-options policy-statement EXPORT-UPSTREAM term 2 then accept
set policy-options policy-statement EXPORT-UPSTREAM term reject then reject
set policy-options policy-statement IMPORT-ALL term 1 from protocol bgp
set policy-options policy-statement IMPORT-ALL term 1 then accept
set policy-options policy-statement IMPORT-ALL term reject then reject
set policy-options policy-statement IMPORT-AS70-IX term 1 from protocol bgp
set policy-options policy-statement IMPORT-AS70-IX term 1 from route-filter 70.0.0.0/22 upto /24
set policy-options policy-statement IMPORT-AS70-IX term 1 then local-preference 150
set policy-options policy-statement IMPORT-AS70-IX term 1 then community set AS70-IX
set policy-options policy-statement IMPORT-AS70-IX term 1 then accept
set policy-options policy-statement IMPORT-AS70-IX term reject then reject
set policy-options policy-statement IMPORT-CLIENTE-60 term 1 from as-path CLIENTE-AS60
set policy-options policy-statement IMPORT-CLIENTE-60 term 1 then community set CLIENTE-60
set policy-options policy-statement IMPORT-CLIENTE-60 term 1 then accept
set policy-options policy-statement IMPORT-CLIENTE-60 term reject then reject
set policy-options policy-statement IMPORT-IX-ALL term 1 from protocol bgp
set policy-options policy-statement IMPORT-IX-ALL term 1 then local-preference 120
set policy-options policy-statement IMPORT-IX-ALL term 1 then community set IX-ALL
set policy-options policy-statement IMPORT-IX-ALL term 1 then accept
set policy-options policy-statement IMPORT-IX-ALL term reject then reject
set policy-options policy-statement IMPORT-PRX-CGNAT term 1 from route-filter 111.1.2.0/24 exact
set policy-options policy-statement IMPORT-PRX-CGNAT term 1 then accept
set policy-options policy-statement IMPORT-PRX-CGNAT term reject then reject
set policy-options community AS70-IX members 10:1070
set policy-options community CLIENTE-60 members 10:60
set policy-options community IX-ALL members 10:123
set policy-options as-path CLIENTE-AS60 ".*60$"
```
LINUX-SRV-DNS
```
ip add add 111.1.1.2/24 dev eth0
ip route add default via 111.1.1.1 dev eth0
nano /etc/hosts
70.0.1.2   web.as70.lab
systemctl restart dnsmasq.service 
```
CISCO-AS30
```
hostname RT-AS30
!
interface Loopback0
 ip address 30.0.1.1 255.255.255.0
!
interface GigabitEthernet0/0
 ip address 10.30.50.1 255.255.255.252
 duplex auto
 speed auto
 media-type rj45
!
interface GigabitEthernet0/1
 no ip address
 duplex auto
 speed auto
 media-type rj45
!
interface GigabitEthernet0/2
 ip address 10.20.30.2 255.255.255.252
 duplex auto
 speed auto
 media-type rj45
!
interface GigabitEthernet0/3
 ip address 10.30.70.1 255.255.255.252
 duplex auto
 speed auto
 media-type rj45
!
router bgp 30
 bgp log-neighbor-changes
 network 30.0.0.0 mask 255.255.252.0
 network 30.0.1.0 mask 255.255.255.0
 neighbor 10.20.30.1 remote-as 20
 neighbor 10.20.30.1 soft-reconfiguration inbound
 neighbor 10.30.50.2 remote-as 50
 neighbor 10.30.50.2 soft-reconfiguration inbound
 neighbor 10.30.70.2 remote-as 70
 neighbor 10.30.70.2 soft-reconfiguration inbound
!
ip route 30.0.0.0 255.255.252.0 Null0
```
VYOS-AS40
```
set interfaces ethernet eth0 address '10.10.40.2/30'
set interfaces ethernet eth1 address '10.40.50.1/30'
set interfaces ethernet eth2 vif 3000 address '123.123.0.40/24'
set interfaces loopback lo address '40.0.1.1/24'
set policy prefix-list ALL rule 10 action 'permit'
set policy prefix-list ALL rule 10 le '24'
set policy prefix-list ALL rule 10 prefix '0.0.0.0/0'
set policy prefix-list PRX-AS40 rule 10 action 'permit'
set policy prefix-list PRX-AS40 rule 10 le '24'
set policy prefix-list PRX-AS40 rule 10 prefix '40.0.0.0/22'
set policy route-map EXPORT-ALL rule 10 action 'permit'
set policy route-map EXPORT-ALL rule 10 match ip address prefix-list 'ALL'
set policy route-map EXPORT-AS40 rule 10 action 'permit'
set policy route-map EXPORT-AS40 rule 10 match ip address prefix-list 'PRX-AS40'
set policy route-map IMPORT-ALL rule 10 action 'permit'
set policy route-map IMPORT-ALL rule 10 match ip address prefix-list 'ALL'
set protocols bgp address-family ipv4-unicast network 40.0.0.0/22
set protocols bgp address-family ipv4-unicast network 40.0.1.0/24
set protocols bgp local-as '40'
set protocols bgp neighbor 10.10.40.1 address-family ipv4-unicast route-map export 'EXPORT-ALL'
set protocols bgp neighbor 10.10.40.1 address-family ipv4-unicast route-map import 'IMPORT-ALL'
set protocols bgp neighbor 10.10.40.1 address-family ipv4-unicast soft-reconfiguration inbound
set protocols bgp neighbor 10.10.40.1 remote-as '10'
set protocols bgp neighbor 10.40.50.2 address-family ipv4-unicast route-map export 'EXPORT-ALL'
set protocols bgp neighbor 10.40.50.2 address-family ipv4-unicast route-map import 'IMPORT-ALL'
set protocols bgp neighbor 10.40.50.2 address-family ipv4-unicast soft-reconfiguration inbound
set protocols bgp neighbor 10.40.50.2 remote-as '50'
set protocols bgp neighbor 123.123.0.254 address-family ipv4-unicast route-map export 'EXPORT-AS40'
set protocols bgp neighbor 123.123.0.254 address-family ipv4-unicast route-map import 'IMPORT-ALL'
set protocols bgp neighbor 123.123.0.254 address-family ipv4-unicast soft-reconfiguration inbound
set protocols bgp neighbor 123.123.0.254 remote-as '123'
set protocols static route 40.0.0.0/22 blackhole
set system host-name 'RT-AS40'
```
CISCO-SW-IX
```
hostname SW-IX
!
vlan 1070
 name BILATERAL-AS10XAS70
!
vlan 3000
 name VLAN-IX-RT-RS
!
interface GigabitEthernet0/0
 no shutdown
 description AS30
 switchport trunk allowed vlan 1070,3000
 switchport trunk encapsulation dot1q
 switchport mode trunk
 negotiation auto
!
interface GigabitEthernet0/1
 no shutdown
 description AS10
 switchport trunk allowed vlan 1070,3000
 switchport trunk encapsulation dot1q
 switchport mode trunk
 negotiation auto
!
interface GigabitEthernet0/2
 no shutdown
 description RS
 switchport access vlan 3000
 switchport mode access
 negotiation auto
!
interface GigabitEthernet0/3
 no shutdown
 description AS40
 switchport trunk allowed vlan 3000
 switchport trunk encapsulation dot1q
 switchport mode trunk
 negotiation auto
!
```
HUAWEI-AS50
```
sysname RT-AS50
#
interface Ethernet1/0/0
 description PTP-AS40
 undo shutdown
 ip address 10.40.50.2 255.255.255.252
 undo dcn
 undo dcn mode vlan
#
interface Ethernet1/0/1
 description PTP-AS30
 undo shutdown  
 ip address 10.30.50.2 255.255.255.252
 undo dcn
 undo dcn mode vlan
#
interface LoopBack0
 ip address 50.0.1.1 255.255.255.255
#
interface NULL0
#
bgp 50
 peer 10.30.50.1 as-number 30
 peer 10.30.50.1 description AS30
 peer 10.40.50.1 as-number 40
 peer 10.40.50.1 description AS40
 #
 ipv4-family unicast
  undo synchronization
  network 50.0.0.0 255.255.252.0
  peer 10.30.50.1 enable
  peer 10.30.50.1 route-policy ALL import
  peer 10.30.50.1 route-policy EXPORT-ALL export
  peer 10.40.50.1 enable
  peer 10.40.50.1 route-policy ALL import
  peer 10.40.50.1 route-policy EXPORT-ALL export
#
undo dcn
#
route-policy ALL permit node 10
 if-match ip-prefix ALL
#
route-policy EXPORT-ALL permit node 10
 if-match ip-prefix ALL
#
ip ip-prefix ALL index 10 permit 0.0.0.0 0 less-equal 24
#
ip route-static 50.0.0.0 255.255.252.0 NULL0
#
```
MIKROTIK-AUTENTICADOR
```
/interface bridge add name=loopback
/interface vlan add interface=ether1 name=pppoe vlan-id=4000
/interface vlan add interface=ether2 name=ptp-cgnat vlan-id=3013
/interface vlan add interface=ether2 name=ptp-rt1 vlan-id=1013
/interface vlan add interface=ether2 name=ptp-rt2 vlan-id=2013
/ip pool add name=cliente-pppoe ranges=100.64.0.1-100.64.0.10
/ppp profile set *0 dns-server=111.1.1.2 local-address=10.10.3.3 remote-address=cliente-pppoe
/routing bgp instance set default as=10
/ip firewall connection tracking set enabled=no
/interface pppoe-server server add disabled=no interface=pppoe one-session-per-host=yes service-name=pppoe-server
/ip address add address=10.10.13.2/30 interface=ptp-rt1 network=10.10.13.0
/ip address add address=10.10.23.2/30 interface=ptp-rt2 network=10.10.23.0
/ip address add address=10.10.34.1/30 interface=ptp-cgnat network=10.10.34.0
/ip address add address=10.10.3.3 interface=loopback network=10.10.3.3
/ip route add distance=1 gateway=10.10.34.2 routing-mark=CGNAT
/ip route add distance=1 dst-address=100.64.0.0/24 type=blackhole
/ip route rule add action=lookup-only-in-table src-address=100.64.0.0/24 table=CGNAT
/ppp secret add name=cliente password=cliente service=pppoe
/routing bgp network add network=100.64.0.0/24
/routing bgp peer add in-filter=IN-DISCARD name=ibgp-cgnat out-filter=OUT remote-address=10.10.34.2 remote-as=10
/routing bgp peer add in-filter=INPUT-DEFAULT name=ibgp-rt1 out-filter=OUT-DISCARD remote-address=10.10.13.1 remote-as=10
/routing bgp peer add in-filter=INPUT2-DEFAULT name=ibgp-rt1 out-filter=OUT-DISCARD remote-address=10.10.23.1 remote-as=10
/routing filter add action=accept chain=INPUT-DEFAULT prefix=0.0.0.0/0 set-bgp-local-pref=110
/routing filter add action=discard chain=INPUT-DEFAULT
/routing filter add action=accept chain=INPUT2-DEFAULT prefix=0.0.0.0/0
/routing filter add action=discard chain=INPUT2-DEFAULT
/routing filter add action=discard chain=IN-DISCARD
/routing filter add action=accept chain=OUT prefix=100.64.0.0/24
/routing filter add action=discard chain=OUT
/routing filter add action=discard chain=OUT-DISCARD
/system identity set name=AUTENTIC
```
MIKROTIK-AS20
```
/routing bgp template set default as=20 input.filter=IMPORT-ALL output.filter-chain=EXPORT-ALL .network=PRX-AS20
/ip address add address=10.20.30.1/30 interface=ether2 network=10.20.30.0
/ip address add address=10.10.20.2/30 interface=ether1 network=10.10.20.0
/ip address add address=20.0.1.1/24 interface=lo network=20.0.1.0
/ip firewall address-list add list=ALL
/ip firewall address-list add list=PRX-AS20
/ip firewall address-list add address=20.0.0.0/22 list=ALL
/ip firewall address-list add address=20.0.0.0/22 list=PRX-AS20
/ip firewall address-list add address=20.0.1.0/24 list=PRX-AS20
/ip firewall address-list add address=20.0.1.0/24 list=ALL
/ip firewall address-list add address=30.0.0.0/22 list=ALL
/ip firewall address-list add address=30.0.1.0/24 list=ALL
/ip firewall address-list add address=60.0.0.0/22 list=ALL
/ip firewall address-list add address=60.0.1.0/24 list=ALL
/ip firewall address-list add address=111.1.0.0/22 list=ALL
/ip firewall address-list add address=111.1.1.0/24 list=ALL
/ip firewall address-list add address=111.1.2.0/24 list=ALL
/ip firewall address-list add address=70.0.0.0/22 list=ALL
/ip firewall address-list add address=70.0.1.0/24 list=ALL
/ip firewall address-list add address=50.0.0.0/22 list=ALL
/ip firewall address-list add address=50.0.1.0/24 list=ALL
/ip firewall address-list add address=40.0.1.0/24 list=ALL
/ip firewall address-list add address=40.0.0.0/22 list=ALL
/ip route add blackhole dst-address=20.0.0.0/22
/routing bgp connection add local.role=ebgp name=PEER-AS30 remote.address=10.20.30.2 .as=30 templates=default
/routing bgp connection add local.role=ebgp name=PEER-AS10 remote.address=10.10.20.1 .as=10 templates=default
/routing filter rule add chain=IMPORT-ALL rule="if (dst in ALL ) {accept; }"
/routing filter rule add chain=EXPORT-ALL rule="if (dst in ALL ) {accept; }"
/system identity set name=RT-AS20
```
BIRD-IX
```
ip add add 123.123.0.254/24 dev eth0

nano /etc/bird/bird.conf 

protocol bgp peer70 { 
description "AS 70"; 
local as 123; 
neighbor 123.123.0.70 as 70; 
rs client;  
import all; 
export all;
}
protocol bgp peer10 { 
description "AS 10"; 
local as 123; 
neighbor 123.123.0.10 as 10; 
rs client;  
import all; 
export all;
}
protocol bgp peer40 { 
description "AS 40"; 
local as 123; 
neighbor 123.123.0.40 as 40; 
rs client;  
import all; 
export all;
}

systemctl restart bird

birdc
```
MIKROTIK-AS60
```
/interface bridge add name=loopback
/interface vlan add interface=ether1 name=PTP-RT1 vlan-id=1111
/interface vlan add interface=ether1 name=PTP-RT2 vlan-id=2222
/routing bgp instance set default as=60
/ip address add address=111.1.0.102/30 interface=PTP-RT1 network=111.1.0.100
/ip address add address=111.1.0.202/30 interface=PTP-RT2 network=111.1.0.200
/ip address add address=60.0.1.1/24 interface=loopback network=60.0.1.0
/ip route add distance=1 dst-address=60.0.0.0/22 type=blackhole
/routing bgp network add network=60.0.0.0/22
/routing bgp network add network=60.0.1.0/24
/routing bgp peer add in-filter=IMPORT-ALL-RT1 name=peer1 out-filter=EXPORT-AS60 remote-address=111.1.0.101 remote-as=10
/routing bgp peer add in-filter=IMPORT-ALL-RT2 name=peer2 out-filter=EXPORT-AS60 remote-address=111.1.0.201 remote-as=10
/routing filter add action=accept chain=IMPORT-ALL-RT1 set-bgp-local-pref=110
/routing filter add action=accept chain=IMPORT-ALL-RT2
/routing filter add chain=EXPORT-AS60 prefix=60.0.0.0/22
/routing filter add chain=EXPORT-AS60 prefix=60.0.1.0/24
/system identity set name=RT-AS60
```
CISCO-SW-PPPOE
```
hostname SW-3-PPPOE
!
vlan 4000 
!
interface GigabitEthernet0/0
 switchport trunk allowed vlan 4000
 switchport trunk encapsulation dot1q
 switchport mode trunk
 negotiation auto
!
interface GigabitEthernet0/1
 switchport access vlan 4000
 switchport mode access
 negotiation auto
!
```
CISCO-AS70
```
hostname RT-AS70
!
ip dhcp pool LAN
 network 70.0.1.0 255.255.255.0
 default-router 70.0.1.1
 dns-server 111.1.1.2
!
no ip domain lookup
!
interface GigabitEthernet0/0
 no shutdown
 description LAN
 ip address 70.0.1.1 255.255.255.0
 duplex auto
 speed auto
 media-type rj45
!
interface GigabitEthernet0/1
 no shutdown
 description PTP-UPSTREAM-AS30
 ip address 10.30.70.2 255.255.255.252
 duplex auto
 speed auto
 media-type rj45
!
interface GigabitEthernet0/2
 no shutdown
 no ip address
 duplex auto
 speed auto
 media-type rj45
!
interface GigabitEthernet0/2.1070
 encapsulation dot1Q 1070
 ip address 10.10.70.2 255.255.255.252
!
interface GigabitEthernet0/2.3000
 description IX-RS
 encapsulation dot1Q 3000
 ip address 123.123.0.70 255.255.255.0
!
router bgp 70
 no bgp enforce-first-as
 bgp log-neighbor-changes
 network 70.0.0.0 mask 255.255.252.0
 network 70.0.1.0 mask 255.255.255.0
 neighbor 10.10.70.1 remote-as 10
 neighbor 10.10.70.1 soft-reconfiguration inbound
 neighbor 10.10.70.1 route-map IMPORT-ALL in
 neighbor 10.10.70.1 route-map EXPORT-AS70 out
 neighbor 10.30.70.1 remote-as 30
 neighbor 10.30.70.1 soft-reconfiguration inbound
 neighbor 10.30.70.1 route-map IMPORT-ALL in
 neighbor 10.30.70.1 route-map EXPORT-AS70 out
 neighbor 123.123.0.254 remote-as 123
 neighbor 123.123.0.254 soft-reconfiguration inbound
 neighbor 123.123.0.254 route-map IMPORT-ALL in
 neighbor 123.123.0.254 route-map EXPORT-AS70 out
!
ip route 70.0.0.0 255.255.252.0 Null0
!
ip prefix-list ALL seq 5 permit 0.0.0.0/0 le 24
!
ip prefix-list PRX-AS70 seq 5 permit 70.0.0.0/22 le 24
!
route-map EXPORT-AS70 permit 10
 match ip address prefix-list PRX-AS70
!
route-map IMPORT-ALL permit 10
 match ip address prefix-list ALL
 ```