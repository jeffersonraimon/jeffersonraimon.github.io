---
title: "29 | Configurando Proxmox com VLAN, Bridge e Bonding (Agregação de Links)"
date: 2026-02-19 00:00:00 -0300
categories: [Tutoriais]
tags: [Proxmox, VLAN, Bridge, Bonding, Linux, Virtualizacao]
image:
  path: assets/img/posts/post-29/capa.png
---

O cenário de Proxmox envolvendo VLANs, bridges e bonding pode gerar algumas dúvidas, principalmente quando estamos lidando com múltiplas interfaces e configurações de rede mais avançadas. Neste tutorial, vou explicar como configurar corretamente esse ambiente, com base em um exemplo prático de configuração de rede para containers LXC — mas vale para VMs também.

Visão Geral do Cenário
Imagine que você tem dois hosts Proxmox (PVE) em cluster, conectados a um switch com trunking de VLANs. Cada host possui três interfaces físicas, e a rede será configurada para funcionar com bonding e bridges para garantir alta disponibilidade e balanceamento de carga.

Abaixo, temos a imagem que resume o funcionamento dessa arquitetura:

![](assets/img/posts/post-29/capa.png)


Passo a Passo da Configuração
1. Criar o Bonding
Primeiramente, crie uma interface de bonding agregando suas interfaces físicas (nesse exemplo, ens3, ens4 e ens5). O bonding vai combinar essas interfaces para aumentar a largura de banda e garantir maior disponibilidade de rede. Importante: não adicione IPs a essas interfaces de bonding — elas são apenas para tráfego de rede.

Existem alguns modos de Bonding que influenciam no balanceamento ou atuar como backup. 

[Veja aqui](https://pve.proxmox.com/wiki/Network_Configuration#sysadmin_network_bond)

![](assets/img/posts/post-29/01.png)


2. Criar a Bridge Linux
Com o bonding criado, o próximo passo é configurar uma Linux Bridge. Essa bridge funcionará como a interface virtual que conecta as VMs ou containers à rede física. Para isso, adicione o bonding criado como a “Bridge Port” da Linux Bridge.

![](assets/img/posts/post-29/02.png)

3. Criar a VLAN Linux
Agora, para adicionar IPs para fins de gerência ou para o cluster, você deve criar uma VLAN Linux. O nome da VLAN precisa seguir a convenção: deve começar com o nome da Linux Bridge e ser seguido pelo número da VLAN, com um ponto no meio. Exemplo: se sua bridge se chama vmbr0, a interface da VLAN pode ser vmbr0.1100 (para a VLAN ID 1100). 

No meu exemplo, configurei duas VLANs:

VLAN 1100 para gerenciamento (MGMT)
VLAN 1200 para o link do cluster Proxmox

![](assets/img/posts/post-29/03.png)

4. Configuração nas VMs ou LXC
Por fim, para as VMs ou containers LXC, a configuração é bem simples. Ao configurar a interface de rede no container ou VM, basta atribuir a bridge correta e a VLAN Tag desejada. No exemplo abaixo, estou utilizando a VLAN 1300, que está associada a um servidor DHCP configurado no roteador. A comunicação entre o container/VM e o servidor DHCP ocorrerá normalmente.

![](assets/img/posts/post-29/04.png)


CLI
Deixo aqui como fica a configuração completa
```
root@pve01:~# cat /etc/network/interfaces
auto lo
iface lo inet loopback

auto ens3
iface ens3 inet manual

auto ens4
iface ens4 inet manual

auto ens5
iface ens5 inet manual

auto bond0
iface bond0 inet manual
        bond-slaves ens3 ens4 ens5
        bond-miimon 100
        bond-mode balance-tlb

auto vmbr0
iface vmbr0 inet manual
        bridge-ports bond0
        bridge-stp off
        bridge-fd 0

auto vmbr0.1100
iface vmbr0.1100 inet static
        address 10.0.137.19/24
        gateway 10.0.137.254

auto vmbr0.1200
iface vmbr0.1200 inet static
        address 10.1.100.1/30

source /etc/network/interfaces.d/*
```
Para aplicar:
```
systemctl restart networking.service
```
Config completa na GUI

![](assets/img/posts/post-29/05.png)

Conclusão
Com esses passos, você consegue configurar um ambiente robusto e eficiente no Proxmox, utilizando VLANs, bridges e bonding para otimizar o desempenho e a segurança da sua infraestrutura.