---
title: "08 | Introdução ao Segment Routing e Implementação Prática com FFRouting"
date: 2024-08-10 00:00:00 -0300
categories: [Artigos]
tags: [SegmentRouting, FRRouting, MPLS]
image:
  path: assets/img/posts/post-08/capa.jpg
---

No mundo dos laboratórios de redes, as máquinas virtuais (VMs) de switches são extremamente importantes para treinamento, desenvolvimento e aprendizagem. Entre as diversas opções disponíveis, o ExtremeXOS, da Extreme Networks, se destaca pela sua versatilidade e robustez em funcionalidades. Comparado a soluções de outros fabricantes, como Cisco e Huawei, o ExtremeXOS oferece uma experiência mais completa e menos restritiva, o que é fundamental para a realização de laboratórios e testes avançados. Apesar de adotar um paradigma e uma abordagem de configuração distintos dos switches Cisco-like, é importante lembrar que os princípios teóricos se aplicam universalmente a todos os fabricantes.

Funcionalidades Avançadas no ExtremeXOS

Uma das principais vantagens da VM de switch ExtremeXOS é a ampla gama de funcionalidades que ela oferece sem restrições significativas. A Extreme Networks proporciona acesso a recursos avançados como EAPS (Ethernet Automatic Protection Switching), VPWS (Virtual Private Wire Service) e VPLS (Virtual Private LAN Service). Essas tecnologias são essenciais para simular cenários complexos de redes e serviços, e o ExtremeXOS permite sua implementação de forma plena em ambientes virtuais.

Limitações em outras soluções

Quando comparado com o ExtremeXOS, outras soluções de fabricantes como Cisco e Huawei mostram algumas limitações que podem impactar a experiência de laboratório.

- Cisco: As VMs de switches Cisco, especialmente no contexto de switches Layer 3, enfrentam restrições significativas. Um exemplo notável é a limitação em criar L2VPNs (Layer 2 Virtual Private Networks) nas VMs. Isso restringe a capacidade de simular e testar configurações de redes privadas virtuais de camada 2, limitando a flexibilidade e a profundidade dos testes possíveis.
- Huawei: As VMs de switches Huawei também apresentam desafios. Algumas imagens de seus switches virtuais têm problemas com o tráfego de VLANs e podem exigir configurações adicionais ou soluções alternativas para funcionar corretamente. Isso pode levar a uma experiência de laboratório menos fluida e mais propensa a problemas técnicos, prejudicando a eficácia do treinamento e o desenvolvimento de habilidades.

Infelizmente a VM do ExtremeXOS pode apresentar algumas limitações relacionadas ao plano de encaminhamento, o que pode afetar a captura e visualização de pacotes usando ferramentas como o Wireshark. Apesar de funcionar normalmente o transporte de vlans em VPWS/VPLS por exemplo, tive problemas com trafego de OSPF nesse transporte.

Alguns exemplos de configurações para se ter um norte
Imagem: [extremexos-32.6.3.126 instalado no PNETLAB](https://github.com/extremenetworks/Virtual_EXOS)

Criação de VLAN e Interface VLAN
``` 
configure vlan default delete ports all //remover a vlan default de todas a portas

create vlan "ptp-sw1-2"
configure vlan ptp-sw1-2 tag 12
configure vlan ptp-sw1-2 add ports 1,2 tagged 

configure vlan ptp-sw1-2 ipaddress 10.0.12.1 255.255.255.252
enable ipforwarding vlan ptp-sw1-2 //necessário para roteamento com OSPF, por exemplo
``` 
Criação de uma Loopback
``` 
create vlan "loopback"
configure vlan loopback tag 1000
enable loopback-mode vlan loopback

configure vlan loopback ipaddress 10.1.1.1 255.255.255.255
enable ipforwarding vlan loopback //necessário para roteamento com OSPF, por exemplo
``` 
OSPF
``` 
configure ospf routerid 10.1.1.1
enable ospf
configure ospf add vlan loopback area 0.0.0.0 
configure ospf add vlan ptp-sw1-2 area 0.0.0.0 
``` 
MPLS
``` 
debug epm enable trial-license // comando necessário para habilitar o MPLS e outros processos

configure mpls add vlan "loopback"
configure mpls add vlan "ptp-sw1-2"
enable mpls vlan "ptp-sw1-2"
enable mpls ldp vlan "ptp-sw1-2"

configure mpls lsr-id 10.1.1.1
enable mpls protocol ldp
enable mpls
``` 
L2VPN VPWS
``` 
disable igmp snooping vlan v100-MK

create l2vpn vpws MK5-6 fec-id-type pseudo-wire 100
configure l2vpn vpws MK5-6 add service vlan v100-MK
configure l2vpn vpws MK5-6 add peer 10.4.4.4 // não precisa criar um "ldp remote" antes, o ExtremeXOS cria automaticamente após setar o peer
``` 
L2VPN VPLS
``` 
disable igmp snooping vlan MKS

create l2vpn vpls MK-VPLS fec-id-type pseudo-wire 200
configure l2vpn vpls MK-VPLS add service vlan MKS
configure l2vpn vpls MK-VPLS add peer 10.3.3.3
configure l2vpn vpls MK-VPLS add peer 10.4.4.4 
``` 
EAPS
``` 
enable eaps
configure eaps fast-convergence on
create eaps EAPS-DOMAIN
configure eaps EAPS-DOMAIN mode master
configure eaps EAPS-DOMAIN primary port 1
configure eaps EAPS-DOMAIN secondary port 2
enable eaps EAPS-DOMAIN
configure eaps EAPS-DOMAIN add protected vlan v100
configure eaps EAPS-DOMAIN add protected vlan v200
configure eaps EAPS-DOMAIN add control vlan EAPS-CONTROL
``` 
Ver tabela MAC
``` 
show fdb
show fdb vpls
``` 
[Mais comandos](https://documentation.extremenetworks.com/exos_31.6/GUID-7D648968-51CD-4E05-828C-8606BD5C0474.shtml)

[Mais artigos sobre](https://medium.com/@daper52)

Conclusão
A VM de switch ExtremeXOS oferece uma solução robusta e completa para profissionais e estudantes que buscam criar e testar cenários de redes avançados. Com recursos como EAPS, VPWS e VPLS totalmente disponíveis, a Extreme Networks proporciona uma plataforma que se destaca em termos de funcionalidade e flexibilidade. Em contraste, as limitações encontradas nas soluções de Cisco e Huawei podem restringir significativamente as capacidades de simulação e laboratório, tornando o ExtremeXOS uma escolha preferencial para aqueles que buscam uma experiência de aprendizado mais abrangente e menos restritiva.

Escolher a ferramenta certa para o laboratório pode fazer toda a diferença na qualidade do aprendizado e na preparação para desafios reais no campo das redes, e a VM de switch ExtremeXOS certamente se mostra como uma opção superior nesse contexto.