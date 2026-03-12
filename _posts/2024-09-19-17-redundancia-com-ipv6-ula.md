---
title: "17 | Redundância com IPv6 ULA"
date: 2024-09-19 00:00:00 -0300
categories: [Tutoriais]
tags: [IPv6, ULA, Redundancia]
image:
  path: assets/img/posts/post-17/capa.png
---

É comum em IPv4 utilizarmos um /30 privado e rotear outro prefixo público (/29, /30 ou /32 por exemplo) apontando para o outro IP privado no cliente. Porém também é possível fazer com IPv6 utilizando os endereços ULA para evitar utilizar dois prefixos GUA na WAN para o mesmo cliente.

O endereço ULA (Unique Local Address) no IPv6 é um tipo de endereço destinado ao uso em redes locais. Ele é equivalente aos endereços privados no IPv4, permitindo comunicação dentro de uma rede sem necessidade de roteamento na internet. Os endereços ULA começam com o prefixo ```fc00::/7```. Diferentemente de seus equivalentes no IPv4, os endereços locais do IPv6 possuem uma parte aleatória de 40 bits, o que os torna únicos. O objetivo dos endereços locais do IPv6 é que, ao conectar duas redes privadas IPv6 — como dois sites privados conectados via VPN — seja muito improvável que ocorram conflitos de endereçamento. Para isso é possível utilizar geradores de IPv6 ULA para obter prefixos únicos. Exemplo: <https://www.unique-local-ipv6.com/>.

Para demonstrar realizei um simples laboratório.

## Topologia

Criei um cenário em que tenho dois equipamentos no AS155. Um roteador de borda Juniper e outro roteador Mikrotik para os clientes dedicados. O cliente dedicado possui um link principal pela rede GPON e outro de backup por enlace de rádio.

![](assets/img/posts/post-17/capa.png)

## Configurações

MK-RT-DEDICADO
```
/ipv6 address add address=2001:abc:100::2 advertise=no interface=ether1
/ipv6 address add address=fd10:1::1/126 advertise=no interface=ether2
/ipv6 address add address=fd20:2::1/126 advertise=no interface=ether3

/routing bgp instance set default as=155 router-id=10.3.3.3
/routing bgp network add network=2001:abc:1000::/56
/routing bgp peer add address-families=ipv6 name=peer1 remote-address=2001:abc:100::1 remote-as=155

/ipv6 route add check-gateway=ping distance=1 dst-address=2001:abc:1000::/56 gateway=fd10:1::2
/ipv6 route add check-gateway=ping distance=2 dst-address=2001:abc:1000::/56 gateway=fd20:2::2
```

Podemos ver com as configurações acima que estou utilizando no enlace Borda x MK-Dedicado endereços GUA normalmente e para o cliente, uso os endereços ULA e roteando o /56 público dele com check-gateway para manter o failover do meu lado. Faço o anuncio apenas para o meu roteador de borda o /56 do cliente.

OBS: Utilizo /126 porque com /127 não ocorreu comunicação.

![](assets/img/posts/post-17/01.png)

MK-CLIENTE
```
/ipv6 address add address=fd10:1::2/126 advertise=no interface=ether1
/ipv6 address add address=fd20:2::2/126 advertise=no interface=ether2
/ipv6 address add eui-64=yes from-pool=pool interface=ether3

/ipv6 route add dst-address=::/0 distance=1 check-gateway=ping gateway=fd10:1::1
/ipv6 route add dst-address=::/0 distance=2 check-gateway=ping gateway=fd20:2::1

/ipv6 pool add name=pool prefix=2001:abc:1000::/56 prefix-length=64
/ipv6 route add distance=1 dst-address=2001:abc:1000::/56 type=unreachable
```

No roteador do cliente quebro o /56 em /64 e adiciono na interface direcionada para a LAN. Na WAN configuro o /126 ULA e aponto a rota default para o gateway. Também tenho configurado o check-gateway para o failover do lado do cliente (pode ser utilizado qualquer outra técnica). Por fim crio uma rota estática do /56 em “blackhole” para evitar loop de roteamento.

![](assets/img/posts/post-17/02.png)

Roteador de Borda
Recebo o prefixo /56 do cliente por meio do MK-DEDICADO (e o prefixo /32 do AS200 para realizar o teste).

![](assets/img/posts/post-17/03.png)

Roteador AS200
Recebo o prefixo /32 do AS100 e anuncio o prefixo /32 do AS200.

![](assets/img/posts/post-17/04.png)

## Testes

PC Cliente

Validado que temos comunicação normal com o servidor no AS200.

![](assets/img/posts/post-17/05.png)
![](assets/img/posts/post-17/06.png)

Servidor WEB

Mesma coisa para o servidor x cliente. Também conseguimos acessar remotamente o MK do cliente com o endereço GUA do /56 dele que foi quebrado em /64 na interface ether3 (LAN).

![](assets/img/posts/post-17/07.png)
![](assets/img/posts/post-17/08.png)

Fim!