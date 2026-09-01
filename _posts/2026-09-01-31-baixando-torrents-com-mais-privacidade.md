---
title: "31 | Baixando Torrents com mais privacidade"
date: 2026-09-01 00:00:00 -0300
categories: [Tutoriais]
tags: [torrent, vpn]
image:
  path: assets/img/posts/post-31/capa.png
---

O BitTorrent funciona de maneira diferente de um download tradicional. Em vez de baixar o arquivo de um único servidor, você participa de uma rede peer-to-peer (P2P), na qual diversos usuários trocam partes do arquivo diretamente entre si.

Por causa desse funcionamento, o endereço IP público dos participantes fica visível para outros peers que estão compartilhando o mesmo torrent. Na prática, terceiros podem observar quais endereços IP estão participando de determinados swarms.


Swarms: é o conjunto de todos os usuários/dispositivos que estão participando do compartilhamento de um determinado arquivo torrent.

Ele inclui principalmente:

Seeders (semeadores): já possuem 100% do arquivo e estão enviando partes para outras pessoas.
Leechers/Peers: estão baixando o arquivo e, enquanto baixam, também podem enviar as partes que já possuem para outros peers.

Existem inclusive serviços públicos como o [I Know What You Download](https://iknowwhatyoudownload.com/) que coletam informações observadas em redes BitTorrent e permitem consultar torrents associados a determinado endereço IP. Isso demonstra como a atividade P2P pode ser correlacionada com um IP público — embora esses dados não sejam prova definitiva de que uma pessoa específica realizou determinado download, especialmente em conexões compartilhadas, CGNAT ou endereços dinâmicos.

Uma VPN adiciona uma camada de privacidade nesse cenário:

```
Sem VPN:

qBittorrent
     │
     ▼
Internet
     │
     ├── Tracker
     ├── DHT
     └── Peers
          │
          └── enxergam seu IP público

Com VPN:

qBittorrent
     │
     ▼
VPN
     │
     ▼
Internet
     │
     ├── Tracker
     ├── DHT
     └── Peers
          │
          └── enxergam o IP da VPN

```

Assim, os participantes da rede BitTorrent deixam de enxergar diretamente o endereço IP público da sua conexão e passam a enxergar o endereço de saída fornecido pelo servidor VPN.

Vale destacar que VPN não torna o uso de BitTorrent anônimo nem altera a legalidade do conteúdo transferido. Ela é uma ferramenta de privacidade e isolamento de tráfego. Também é importante escolher um provedor que permita tráfego P2P e entender suas políticas de privacidade e retenção de dados.

Neste artigo, vamos fazer justamente isso sem colocar o servidor Linux inteiro atrás da VPN: somente o container do qBittorrent utilizará a Kaspersky VPN, através de WireGuard e Gluetun.


A arquitetura final fica assim:
```
                    ┌─────────────────────────┐
                    │      Servidor Linux     │
                    │                         │
LAN ── :8080 ──────►│ Gluetun                 │
                    │    ▲                    │
                    │    │ network namespace  │
                    │    │                    │
                    │ qBittorrent             │
                    └────┬────────────────────┘
                         │
                         │ WireGuard
                         ▼
                  ┌──────────────┐
                  │ Kaspersky VPN│
                  └──────┬───────┘
                         │
                         ▼
                      Internet

```

## Por que usar o Gluetun?

Uma possibilidade seria instalar a VPN diretamente no sistema operacional. O problema é que isso pode acabar roteando outros serviços do servidor pela VPN.

O Gluetun funciona como um gateway VPN em container e suporta WireGuard e OpenVPN.

A ideia é fazer o qBittorrent compartilhar o namespace de rede do Gluetun:
```
network_mode: "service:gluetun"
```
Com isso, o qBittorrent não possui uma conexão Docker independente para acessar a Internet.

Temos:
```
qBittorrent → Gluetun → WireGuard → Kaspersky → Internet
```
Enquanto os demais serviços continuam:
```
Outros containers → conexão normal → Internet
```
Além disso, o firewall do Gluetun funciona como uma camada de proteção caso o túnel VPN deixe de funcionar.

# 1. Obtendo a configuração WireGuard

A Kaspersky permite utilizar sua VPN por meio de clientes VPN de terceiros, incluindo WireGuard.

É necessário gerar/obter o arquivo de configuração WireGuard pela sua conta da Kaspersky. As instruções oficiais estão disponíveis [aqui](https://support.kaspersky.com.br/my-kaspersky/227551)

# Configuração da Kaspersky VPN em clientes de terceiros

O arquivo terá uma estrutura semelhante a:
```
[Interface]
PrivateKey = SUA_CHAVE_PRIVADA
Address = 10.x.x.x/32
DNS = x.x.x.x

[Peer]
PublicKey = CHAVE_PUBLICA
AllowedIPs = 0.0.0.0/0
Endpoint = servidor-da-kaspersky:porta
```
Importante: nunca publique ou compartilhe o valor de PrivateKey.

# 2. Habilitando TUN no Linux

O Gluetun precisa do dispositivo:
```
/dev/net/tun
```
Verifique:
```
ls -l /dev/net/tun
```
No meu caso inicialmente recebi:
```
ls: /dev/net/tun: No such file or directory
```
Foi necessário carregar o módulo:
```
sudo modprobe tun
```
Depois:
```
ls -l /dev/net/tun
```
passou a retornar algo semelhante a:
```
crw-rw-rw- 1 root netdev 10, 200 /dev/net/tun
```
Para carregar automaticamente após reinicializações:
```
echo tun | sudo tee /etc/modules-load.d/tun.conf
```
# 3. Preparando o arquivo WireGuard

Crie o diretório:
```
sudo mkdir -p /srv/config/gluetun/wireguard
```
Coloque o arquivo como:
```
/srv/config/gluetun/wireguard/wg0.conf
```
E proteja a chave:
```
sudo chmod 600 /srv/config/gluetun/wireguard/wg0.conf
```
A estrutura ficará:
```
/srv/config/gluetun/
└── wireguard/
    └── wg0.conf
```
# 4. Atenção ao Endpoint da Kaspersky

Aqui encontrei uma particularidade importante.

Meu arquivo da Kaspersky utilizava um hostname no Endpoint, semelhante a:
```
Endpoint = sx022001-ikev.dnsdialer.com:PORTA
```
Ao iniciar o Gluetun, ele retornou:
```
ERROR reading VPN settings:
WIREGUARD_ENDPOINT_IP:
ParseAddr("sx022001-ikev.dnsdialer.com"):
unexpected character
```
O Gluetun esperava um endereço IP nesse campo.

Resolvi o hostname:
```
dig +short A sx022001-ikev.dnsdialer.com
```
E substituí no wg0.conf:
```
Endpoint = IP_RESOLVIDO:PORTA
```
Mantendo exatamente a porta fornecida pela Kaspersky.

Guarde o hostname original, pois o endereço IP associado ao servidor pode mudar no futuro.

# 5. Docker Compose

Meu qBittorrent originalmente utilizava:
```
network_mode: "host"
```
Para obrigá-lo a utilizar a VPN, removemos isso e fazemos o container compartilhar a rede do Gluetun.

O compose fica:
```
services:

  gluetun:
    image: qmcgaw/gluetun:latest
    container_name: gluetun

    cap_add:
      - NET_ADMIN

    devices:
      - /dev/net/tun:/dev/net/tun

    environment:
      - TZ=America/Bahia
      - VPN_SERVICE_PROVIDER=custom
      - VPN_TYPE=wireguard

    volumes:
      - /srv/config/gluetun:/gluetun

    ports:
      # qBittorrent WebUI
      - "8080:8080"

      # BitTorrent
      - "6881:6881"
      - "6881:6881/udp"

    restart: unless-stopped


  qbittorrent:
    image: ghcr.io/hotio/qbittorrent
    container_name: qbittorrent

    network_mode: "service:gluetun"

    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Bahia
      - WEBUI_PORT=8080

    volumes:
      - /srv/config/qbittorrent:/config
      - /srv/media/hd-media/torrents:/downloads

    depends_on:
      gluetun:
        condition: service_healthy

    restart: unless-stopped
```
Observe um detalhe importante: as portas agora são publicadas pelo Gluetun, e não pelo qBittorrent.

Isso ocorre porque:
```
network_mode: "service:gluetun"
```
faz com que os dois containers compartilhem o mesmo namespace de rede.

# 6. Subindo os containers

Execute:
```
docker compose up -d
```
Confira:
```
docker ps
```
E acompanhe o Gluetun:
```
docker logs -f gluetun
```
A WebUI do qBittorrent continuará disponível pela LAN:
```
http://IP-DO-SERVIDOR:8080
```
Ou seja, utilizar a VPN para o tráfego externo não impede o gerenciamento do qBittorrent pela rede local.

# 7. Verificando se o qBittorrent está realmente preso ao Gluetun

Execute:
```
docker inspect qbittorrent --format '{{.HostConfig.NetworkMode}}'
```
No meu caso retornou:
```
container:43011604a5d6...
```
Isso confirma que o qBittorrent está utilizando o namespace de rede de outro container — nesse caso, o Gluetun.

# 8. Comparando o IP do servidor e da VPN

Primeiro consulte o IP normal do servidor:
```
curl -4 https://icanhazip.com
```
Depois consulte através do Gluetun:
```
docker exec gluetun wget -qO- https://ipinfo.io/ip
```
No meu teste, o Gluetun retornou:
```
188.XXX.XXX.135
```
enquanto o servidor apresentava outro IP público.

Isso confirma:
```
Servidor ───────────────────→ Internet
             IP real

qBittorrent → Gluetun
                 ↓
             WireGuard
                 ↓
             Kaspersky
                 ↓
              Internet
             IP da VPN
```
# 9. Testando o kill switch

Também é importante garantir que o qBittorrent não utilize acidentalmente o IP normal caso a VPN fique indisponível.

Como ele utiliza:
```
network_mode: "service:gluetun"
```
podemos fazer um teste simples:
```
docker stop gluetun
```
O qBittorrent deve perder a conectividade em vez de passar automaticamente para a conexão normal do servidor.

Depois:
```
docker start gluetun
```
Esse isolamento é uma das principais vantagens de utilizar essa arquitetura.

# 10. Validando o IP realmente visto pelo BitTorrent

O teste com curl confirma o IP do Gluetun, mas eu queria validar algo mais importante:

*qual endereço IP um tracker realmente enxerga vindo do qBittorrent?*

Para isso utilizei o IPLeak.net

Na seção Torrent Address detection, o site fornece um Magnet Link específico para teste.

Adicionei esse magnet diretamente ao qBittorrent e aguardei alguns segundos.

O resultado foi:
```
Torrent Address detection

IP:   188.XXX.XXX.135
Port: 6881
```
Exatamente o mesmo endereço público apresentado pelo Gluetun.

Portanto, conseguimos validar o caminho completo:
```
qBittorrent
     │
     ▼
  Gluetun
     │
     ▼
WireGuard
     │
     ▼
Kaspersky VPN
     │
     ▼
188.241.120.135
     │
     ▼
Tracker / Peers
```
O IP público normal do servidor não apareceu no teste BitTorrent.

Network Interface do qBittorrent: tun0 ou Any?

O qBittorrent permite selecionar uma interface específica em:
```
Settings
→ Advanced
→ Network Interface
```
Poderíamos selecionar tun0, mas neste cenário preferi deixar:
```
Any interface
```
Isso porque o isolamento já ocorre em uma camada anterior.

O qBittorrent inteiro está dentro do namespace de rede do Gluetun:
```
qBittorrent
     │
     └── não possui saída independente
                 │
                 ▼
              Gluetun
              ├── tun0 → VPN
              └── firewall
```
Assim, deixamos o Gluetun responsável pelo roteamento e pelo kill switch, enquanto evitamos possíveis problemas de binding da WebUI.

# Resultado final

Depois da configuração, temos três características importantes:

- ✔ Outros serviços do servidor continuam usando o IP normal

- ✔ qBittorrent utiliza o IP da Kaspersky VPN

- ✔ Se a VPN ficar indisponível, o qBittorrent não deve fazer fallback
  para a conexão normal

E o teste externo confirmou que o endereço observado pelo tráfego BitTorrent era realmente o IP da VPN.

Para quem executa vários serviços em um mesmo servidor Linux, essa abordagem é particularmente interessante porque não é necessário colocar o host inteiro atrás da VPN.

O Docker fornece o isolamento, o Gluetun controla a conectividade VPN e o qBittorrent simplesmente utiliza o namespace de rede fornecido por ele.
