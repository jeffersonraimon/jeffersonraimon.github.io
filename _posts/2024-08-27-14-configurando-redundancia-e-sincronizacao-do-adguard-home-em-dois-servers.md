---
title: "14 | Configurando Redundância e Sincronização do AdGuard Home em dois Servers"
date: 2024-08-27 00:00:00 -0300
categories: [Tutoriais]
tags: [AdGuard, DNS, Redundancia]
image:
  path: assets/img/posts/post-14/capa.png
---

Neste tutorial, vou mostrar como configurei meu homelab para garantir alta disponibilidade e sincronização de listas de bloqueios DNS utilizando o AdGuard Home. Com um Orange Pi 3 LTS e um LXC Proxmox, configurei duas instâncias do AdGuard Home com redundância usando Keepalived (VRRP) e sincronização de listas com o Docker adguardhome-sync. Esse setup garante que, mesmo que uma instância do AdGuard Home falhe, o serviço continuará funcionando sem interrupções e que só precise adicionar/alterar listas e reescritas DNS em uma instância e será replicada automaticamente.

Topologia resumida

## Servidores do Ambiente

### Server 1

| Propriedade         | Valor                    |
| ------------------- | ------------------------ |
| Tipo                | LXC Container – Proxmox  |
| Hardware            | Mini PC – Beelink Mini S |
| IPv4                | 10.1.10.10               |
| IPv6                | fd10:1:10::10            |
| Sistema Operacional | Ubuntu 23.10             |

---

### Server 2

| Propriedade | Valor           |
| ----------- | --------------- |
| Hardware    | Orange Pi 3 LTS |
| IPv4        | 10.1.10.2       |
| IPv6        | fd10:1:10::2    |
|S.O          |Armbian 24.5.1 Bookworm. Kernel 6.6.36-current-sunxi64 |


IPv4 Virtual entre os servidores: 10.1.10.55

IPv6 Virtual entre os servidores: fd10:1:10::55

## Instalação e configuração

Adguard Home em ambos os servidores
```
curl -s -S -L https://raw.githubusercontent.com/AdguardTeam/AdGuardHome/master/scripts/install.sh | sh -s -- -v
```
Mais detalhes: <https://github.com/AdguardTeam/AdGuardHome>

Caso desejar, instale e configure o Unbound, conforme tutorial do pi-hole: <https://docs.pi-hole.net/guides/dns/unbound/>

```
/etc/unbound/unbound.conf.d/adguardhome.conf

server:
    verbosity: 0
    interface: 127.0.0.1
    interface: ::1
    port: 5335
    do-ip4: yes
    do-udp: yes
    do-tcp: yes
    do-ip6: yes
    prefer-ip6: yes
    harden-glue: yes
    harden-dnssec-stripped: yes
    use-caps-for-id: no
    edns-buffer-size: 1232
    prefetch: yes
    num-threads: 1
    so-rcvbuf: 1m
    private-address: 192.168.0.0/16
    private-address: 169.254.0.0/16
    private-address: 172.16.0.0/12
    private-address: 10.0.0.0/8
    private-address: fd00::/8
    private-address: fe80::/10
```    
Keepalived
```
sudo apt-get install keepalived
```
Crie/edite o arquivo de configuração ```/etc/keepalived/keepalived.conf``` em ambos os dispositivos:

Orange Pi
```
global_defs {
  router_id adguard2
}

vrrp_instance ADGUARDv4 {
  state MASTER
  interface eth0x
  virtual_router_id 55
  priority 100
  advert_int 1
  track_src_ip
  unicast_peer {
    10.1.10.10
  }

  authentication {
    auth_type PASS
    auth_pass TUTORIALX
  }

  virtual_ipaddress {
    10.1.10.55/26
  }
}

vrrp_instance ADGUARDv6 {
  state MASTER
  interface eth0x
  virtual_router_id 55
  priority 100
  advert_int 1

  virtual_ipaddress {
    fd10:1:10::55/64
  }
}
```
LXC
```
global_defs {
  router_id adguard1
}

vrrp_instance ADGUARDv4 {
  state MASTER
  interface eth0
  virtual_router_id 55
  priority 150
  advert_int 1
  track_src_ip
  unicast_peer {
    10.1.10.2
  }

  authentication {
    auth_type PASS
    auth_pass TUTORIALX
  }

  virtual_ipaddress {
    10.1.10.55/26
  }
}

vrrp_instance ADGUARDv6 {
  state MASTER
  interface eth0
  virtual_router_id 55
  priority 150
  advert_int 1

  virtual_ipaddress {
    fd10:1:10::55/64
  }
}
```
Estou definindo o LXC como principal (conforme prioridade maior)

Reinicie o keepalived
```
systemctl restart keepalived
```
Adguard Home Sync

Use o docker compose criado pelo bakito: <https://github.com/bakito/adguardhome-sync> no LXC Container
```
---
version: "2.1"
services:
  adguardhome-sync:
    image: quay.io/bakito/adguardhome-sync
    container_name: adguardhome-sync
    command: run
    environment:
      - ORIGIN_URL=http://10.1.10.10:8079 
      - ORIGIN_USERNAME=user
      - ORIGIN_PASSWORD=password
      - REPLICA_URL=http://10.1.10.2:8079 
      - REPLICA_USERNAME=user
      - REPLICA_PASSWORD=password
      - CRON=0 */2 * * * # run every 1 minute
      - RUNONSTART=true
    ports:
      - 9999:8080
    restart: unless-stopped
```
Como o meu Adguard Home no LXC é o principal, defino ele como o Origin e o Replica o do Orange Pi. Qualquer alteração que eu fizer no Origin, será replicado no secundário.


Agora é só configurar na sua rede o servidor DNS com o IP Virtual e pronto 🙂

Bônus
Como utilizo uma RB750Gr3 com RouterOS v7, posso configurar o Netwatch para monitorar os IPs dos servidores na porta TCP 53. Dependendo do status (UP ou DOWN) desses IPs, posso ajustar as configurações de DNS no DHCP/ND. Se o status estiver UP, os meus servidores DNS serão configurados; caso esteja DOWN, o DNS recursivo do meu provedor será utilizado.



Também é possivel adicionar um template no zabbix para notificações:

<https://github.com/jeffersonraimon/keepalived-zabbix-template>

Conclusão

Com este setup, você terá uma configuração do AdGuard Home em seu homelab, com alta disponibilidade e sincronização automática das configurações de bloqueio/reescritas DNS. Assim, mesmo em caso de falha de hardware, seu DNS continuará funcionando sem problemas.